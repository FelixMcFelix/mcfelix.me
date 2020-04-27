+++
title = "(Almost) Lockless Stream Buffering"
date = 2020-04-27T00:00:00+01:00
description = "Sharing (and saving) bytestreams effectively."
image = ""
tags = ["Rust", "Discord", "Audio", "Concurrency"]
categories = ["Development"]

[[ resources ]]
  name = "header"
  src = "images/header.JPG"
  title = "Japanese pond and stream at the UC Berkeley Botanical Gardens."
+++

Recently, I've been working on retooling the audio processing code for [serenity](https://github.com/serenity-rs/serenity), a Discord bot library.
Adding features like looping, seeking, and shared resources between calls is made difficult when all input data arrives over pipes from `ffmpeg` and similar decoders.
Due to this, I've designed a thread-safe shared stream buffer intended to lock only on accessing and storing new data.

<!--more-->

I wanted to write this up mainly because the data structure needs finer-grained control over shared access and locking than Rust can provide.
It requires `unsafe` code to be expressed performantly, and so demands explanation separate from the audio-specific modifications.
However, I also think the technique could be useful to others.
To keep things simple, I'll describe structs and key fields in code, but leave the algorithmic details to text.

(The actual implementation with many audio-specific modifications [can be found here](https://github.com/FelixMcFelix/serenity/blob/voice-rework/src/voice/streamer.rs#L622-L1641) if needed.)

## High-level introduction

Before I dive into details, the method essentially works by the following:
* Handles contain information about their place in the stream, how they are using that stream, and a reference to a shared store.
* New data is placed into a rope of byte segments.
* Threads only ever lock once they try to read *past* the current stored length. [This is safe for various reasons]({{< relref "#the-critical-section" >}}).
* On stream completion, all rope segments are copied into a single buffer.
* Reads from a finished stream come from a single, contiguous block of data.

Reading from this store can be condensed down to the following algorithm pseudocode:
```python
# Assume we know how many bytes are
# currently stored as backing_len

# main method: reads bytes from stream into buf transparently
# from pos, where reads last came from location.
def read(pos, location, buf):
	if finalised:
		promote handle location to use backing store
	
	if finalised or read is in stored region:
		read_amt = min(buf.len, backing_len - pos)
		local_read(pos, location, buf, read_amt)
	else:
		read = 0
		while read != buf.len or finalised
				and backing_len == pos + read:
			# Note: any thread may change backing_len
			# at any time.
			space = backing_len - pos - read

			if space == 0:
				acquire lock
				recalculate space

				if space == 0:
					fill_from_source(buf.len - read)
					recalculate space
					try to promote location

				# No action if there are now
				# available bytes, read those instead.

				drop lock

			if space > 0:
				local_read(pos + read, location,
					buf[read..], space)

# move some bytes into the target buffer, either
# from the rope or single store
def local_read(pos, location, buf, count):
	if location is backing store:
		read from backing_store[pos..] into buf
	else:
		walk rope until segment containing pos
		fill buf using segment[pos-seg_start..]
		count -= amount of bytes read
		repeat on next segments until count == 0

# add new bytes to rope, possibly finalise
def fill_from_source(to_add):
	remaining = to_add
	while remaining > 0 and stream not done:
		if no space in last segment:
			append new segment to rope
		else:
			read bytes from stream
			to_add -= amount of bytes read

			# allow other streams to access these bytes
			# without lock
			backing_len += amount of bytes read

	# finalisation
	if stream is done:
		allocate large buffer of size backing_len
		for el in rope:
			copy el into buffer[el.start_pos..]
		self.finalised = true

```

At a high level, I hope that this is relatively simple.
In practice, of course, implementing this safely and decently complicates matters.
Now we can get into trickier details of an actual implementation; most of what follows discusses this in the context of Rust.

## Functional requirements
Why should we implement the above in the first place?
Due to our heavy use of `youtube-dl`, `ffmpeg`, and other external streaming audio converters, we have a [Readers-Writers problem](https://en.wikipedia.org/wiki/Readers%E2%80%93writers_problem) with the following constraints:
* Convert a stream into an in-memory, seekable representation.
* Allow safe, shared access to the stored bytestream by many handles.
* Reads past the end of the buffer access an underlying [`Read`](https://doc.rust-lang.org/std/io/trait.Read.html) object, and new data must be stored for other handles to access.
* Minimal locking: no read to an already buffered segment should require a lock.
* Make only small, piecewise allocations (without reallocation).

To meet these goals, we require that each handle has shared mutable access to the byte source and storage.
Due to the nature of the critical section, we need to take control of (im)mutability checking ourselves and assure the compiler that what we're doing is, in fact, sound:
```rust
struct SharedStore {
    raw: UnsafeCell<RawStore>,
}

unsafe impl Send for SharedStore {}
unsafe impl Sync for SharedStore {}
```
This *can* be handled by safe code, albeit slowly, and with a lot of starvation.
In the na&iuml;ve case, shared access of a central buffer can be handled safely by an [`RwLock`](https://docs.rs/parking_lot/0.10.2/parking_lot/type.RwLock.html).
This comes at a cost, however.
Every read requires a read-lock (regardless of whether the stream is finalised), while a single active write-lock prevents *all* reads, even of data which is known to be safe to access.
Likewise, active readers may prevent new data from being stored.

## User handles
Handles require a pointer to the shared store, their current position in the bytestream, and knowledge of which type of internal storage they will be pulling bytes from.

```rust
struct ByteCache {
    core: Arc<SharedStore>,
    pos: usize,
    loc: CacheReadLocation,
}

enum CacheReadLocation {
	Rope(Arc<()>),
    Backed,
}
```
All handles know whether they are using a piecemeal, incrementally allocated store (shown below) or a finalised buffer.
As described in the [basic algorithm above]({{<relref "#high-level-introduction">}}), reads inform the `ByteCache` on whether to upgrade `loc` from `Rope(_)` to `Backed`.
Dropping the `CacheReadLocation` causes a shared counter to decrement, telling the backing store when it can safely deallocate the rope data structure.
I'll explain why below, but `ByteCache` must have a custom `impl Drop` where it attempts to remove the rope from storage, identical to an upgrade operation.

## Handling progressive allocation
The key for allocating (and ensuring safe concurrent access) is to employ a *rope* data structure based on intrusive [`LinkedList`](https://doc.rust-lang.org/std/collections/struct.LinkedList.html)s.
As more space is required, new segments are allocated progressively.
Since this has an O(n) lookup cost, once the underlying stream finishes all data should be moved to a single contiguous buffer.

```rust
struct RawStore {
	len: usize,
	finalised: bool,

	backing_store: Option<Vec<u8>,
	rope: Option<Arc<LinkedList<Chunk>>>,
	rope_users: Arc<()>,
}

struct Chunk {
    data: Vec<u8>,
    start_pos: usize,
}
```

The cache's `rope` is initialised based on either a length hint (if available), or an empty chunk of length `chunk_len`.
As more capacity is required, new chunks of length `chunk_len` are added to the back.
Once the underlying `Read` object returns `Ok(0)` or any error which isn't `Interrupted`, we know that the stream has ended, and may initialise and populate the backing store using chunk data (while preventing any reads which exceed `len`).

Installing the backing store efficiently is considerably more `unsafe` if we only have one chunk, because we can now include a new optimisation.
Optimal behaviour (no copies and no new allocations) requires that we temporarily alias the `Vec<u8>` between the `rope` and `backing_store`.
While it is safe to reconstruct a new `Vec` from identical parts (and we are certain that the new `Vec` has an identical lifetime because it never leaves the `RawStore`), we've set up a contract where the rope `Vec`'s heap buffer must be leaked rather than dropped by the program to prevent a double free.

### Management
This brings us onto how we actually manage the life-cycle of these rope structures.
Even if the data structure is finalised, the `rope` must remain alive so long as any handle could still be reading from it: `CacheReadLocation::Rope(_)` is a strong reference to `rope_users`, which we'll be using as a sentinel for this purpose.
On `Drop` or upgrade of a `ByteCache`, the store checks whether only one reference remains, and attempts to acquire the lock.
* If the lock was unavailable, the upgrade happens and the store is unchanged. Because there is always a chance that several handles could be upgraded at the same time, there is a significant risk that many of them could concurrently see a strong count of 1, and attempt to perform the same operation if unguarded.
* If the lock is acquired, then the rope is deallocated. In the aliased 1-chunk case, this rope chunk's `data` store is leaked to keep it alive in `backing_store`.

The `impl Drop` of `ByteCache` is particularly important when we choose to alias the data store.
Consider the case where we have many `Backed` handles, and one `Rope` handle who never conducts any further reads after finalisation.
When all `Backed` handles are dropped, if the `Rope` is dropped afterwards then the handle must decrement the rope counter (by "upgrading" to `Backed`) and explicitly forget one instance of the aliased `Vec` to prevent a double free.

### Design choices
Given all the talk of performance, the choice of a [`LinkedList`](https://doc.rust-lang.org/std/collections/struct.LinkedList.html) rather than a [`VecDeque`](https://doc.rust-lang.org/std/collections/struct.VecDeque.html) will probably seem strange to most people.
Isn't `VecDeque` a faster choice, recommended by the standard library?
Once again, it comes down to granularity of control when we need to add additional elements.
While `VecDeque` offers O(1) lookup, insertion can very well trigger a full reallocation as the length hint may be incorrect, when the `Vec`-backed list runs out of capacity and resizes.
Beyond the obvious cost of a full copy on every reallocation, this will also invalidate references to the list of `Chunk`s.
This forces the use of a either an `RwLock` around the rope (leading to the aforementioned lock costs and starvation risks), or `Arc`ing every `Chunk` at the cost of extra indirection and the overhead of reference counting.

Separating the `Arc` from the data it concerns, while unorthodox, serves a few purposes:
* Nullifying a layer of indirection,
* Preventing us from needing another [`UnsafeCell`](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html) to share mutability of the list,
* Still ensures automatic decrement on drop.

## The critical section
The critical section concerns a few key operations, and is triggered in only two situations: adding new bytes to the store, or deallocating the rope.
These operations are:
* Modifying the stored length,
* Adding new chunks to the rope,
* Modifying or reading from the underlying byte source (and any encoder/stream transformation state),
* Removal and deallocation of the rope.

Because many of these involve modifying state which other threads will be reading, I'll walk through the most natural observations.

### How do we know it's safe to access `len` even if it's being written to by another thread?
At any point, `len` states that the backing store contains *at least that many bytes*.
By design, the bytestream will never be modified once stored, so reads are guaranteed to be consistent and valid up to `len`.
Additionally, modifications to `len` will only ever increase its value, so bytes which are valid to read will always remain valid.
A handle may not be able to fill its buffer in one call, but it *will* attempt reading/storage until the output buffer is filled or the stream finishes.
In the event that neither condition is met, the handle either adds more bytes itself or blocks until more are available.

### How can multiple handles safely walk a linked list under modification?
Because the start and end of the region to be read from the rope depend only on `RawStore.len`, and not on any `Chunk`'s `data.len()`, we know that no modifications to a `Chunk`'s store data length will affect the bytes we want to read.
Equally, we know that adding new data to `Chunk`s will never cause a reallocation or change `start_pos`, only a modifying the length value of the tail node.
Accessing the referred data is then always safe, because the segment buffer pointers will not change.
As far as access to the list itself is concerned, new chunks only change the tail pointer and the next pointer in the prior tail element, while all walking/iteration occurs from the front of the list.

Finding the correct `Chunk` relies upon both `start_pos` and `data.len()` for every rope element.
`Chunk`s which aren't at the tail of the rope are guaranteed to remain unchanged.
When new bytes are being stored in the tail element of the list, the `Vec` is resized to match its capacity, bytes are read in, and the `Vec` is resized to match the true written amount.
As `len` is incremented after the buffer shrinks to its true size, no invalid bytes may be accessed, so the tail element's length will never decrease below the value it had when `len` was last observed.
However, the amount of bytes from `pos` that a handle wishes to read is computed using `len` -- while more bytes or chunks could have been stored in the interim, accesses to existing data aren't invalidated.


## Further improvements
There are some additional changes and modifications which can be made depending on the use case, or for blanket performance improvements.
* Spawn a new thread for finalisation.
  * This ensures that the thread which finishes a stream doesn't hold up the caller, but slightly complicates the logic for preventing reads past `backing_len`.
* Subtraction on atomics to remove the rope, rather than relying upon an `Arc`.
  * We lose the automatic decrement on drop, but since we have a custom `impl Drop` the mechanisms are very similar.
  * Most importantly, this removes the need to lock because it's immediately clear (through [`fetch_sub`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicUsize.html#method.fetch_sub)) which handle was responsible for reducing the reference count from 2 to 1.
* Store pointers to rope elements in handles, to prevent walking the rope on every access before finalisation.
  * Unfortunately, this requires more `unsafe` code.
  We know that it *is* safe to store such pointers given that `Chunk`s will never move, such pointers would be contained in a `Chained` (and thus remain live as long as the referred rope), and control shared mutable access in the same manner.
  * Currently, this is blocked on [LinkedList cursors](https://github.com/rust-lang/rust/issues/58533) stabilising, or moving to an external library such as [Amanieu's intrusive collections](https://docs.rs/intrusive-collections/).
  Standard library `IterMut`s store the list's length at their time of creation, and will not proceed past this point.

## Conclusion
Thanks for reading!
Hopefully, this has given you some useful insight into solving this particular concurrent buffering problem and how we're solving it.
I'm unsure whether the above has been documented elsewhere: if you know, please feel free to [contact me](mailto:kyleandrew.simpson@gmail.com).
Likewise, if there corrections to be made feel free to request any changes on [GitHub](https://github.com/felixmcfelix/mcfelix.me).