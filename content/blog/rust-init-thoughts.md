+++
title = "Felyne Bot, and Initial Thoughts on Rust"
date = 2018-02-27T22:41:21Z
description = "A language I want to love."
image = "blog/felynes.png"
tags = ["Rust", "Discord", "Monster Hunter", "Bots"]
categories = ["Development", "Programming Languages", "Gaming"]
draft = false
+++

Having been bereft of a way to annoy my fellow Discord users (and itching to learn a new language), I've found myself experimenting with [Rust](https://www.rust-lang.org).
It's been fun and tiring writing [Felyne bot](https://github.com/FelixMcFelix/felyne-bot) in a new language, and I've gotten some more language experience by helping develop [Serenity](https://github.com/zeyla/serenity).

<!--more-->

## Introduction

While intended as a *safe systems programming language*, much of the focus from the community's marketing has been on its ergonomics and ease-of-use while inhabiting very much the same sphere as C.
It caught my interest; I'd been meaning to study and apply the language after learning about region-based memory management this time last year.

On another note, my friends on [Discord](https://discord.gg) (and the groups we tend to frequent) enjoy playing the old Monster Hunter games online.
*Felyne* teammates are a key part of the series, a race of friendly fighting cats who excel at two things: distracting the monsters who obviously want to eat you, and making a hell of a lot of noise.
Not just any noise, mind you.
It's a wonderful, mildly bitcrushed, repetitive, grating, **awfully** loud, and yet strangely comfy experience that only PS2/PSP-era games offer.

So, I wanted to bring their charm, and the charm of the Monster Hunter world to the unenlightened!
I had a language and a project picked out.

To cook up a voice bot I found a pretty decent looking library, [Serenity](https://github.com/zeyla/serenity), which I've been contributing and adding to.
It's currently the most-developed library in the Rust community for Discord bots I've come across, although the voice functionality needed a little love to be where I needed it to be.
I don't often get to work with audio code at the sample level, so working on these aspects of the library in parallel with [Felyne bot](https://github.com/FelixMcFelix/felyne-bot) was worth it.

## The Language

I feel I should start with the good parts of the language (although, I must disclose I haven't tried to toy with macros).

---

Bringing the program model back down to *structs* and *Traits* (like C, except with the kind of generic type reasoning we all love) is very close to the level of abstraction I like to work with.
It's a move away from a certain Object-Oriented dogma that's been floating around for a long time now, but keeping the higher-kinded reasoning (and quality of life) that exists in most languages today.
It's very much in-line with the whole 'zero-cost abstraction' philosophy the devs are going for, and I think it's more than enough to encode fairly rich relationships.

Coming from what I've seen in C, C++, Java and the like, Rust's compiler error messages are a god-send.
Not that I like having my work picked apart, but the clarity and pointedness of the messages themselves is so far above and beyond what other compiled languages really have to offer.
Too often are the days when a relatively simple error in C++ (for instance) leads to pages upon pages of useless information spewing forth from some template code violation.
It's *not* something I want to be spending the time to unpack, and Rust's compiler generally knows how to help you along the path to a working program -- the borrow checker is notoriously harsh, after all!

What I like most, however, are the advances in syntax simplicity and expressiveness over C and C++, which I imagine are its main competitors.
It's freeing to have a language that operates at this level yet distinctly *feels* like a modern language, without carrying around the decaying legacy of prior versions, C++-style.
Enumerations are great, lambdas are great, tuple returns, channels, intelligent destructuring: all great.
Even if it is missing [one of my favourite features from C](https://eli.thegreenplace.net/2011/02/15/array-initialization-with-enum-indices-in-c-but-not-c).

---

[//]: # (?? Issues? Infancy/abandomnent of libraries and sys-wrappers, language wants to fight you at every turn... Build times. Option access on structs, and the compiler not giving helpful messages.)

One of the largest gripes I have is symptomatic of how many people may well have approached the language (i.e., as a curio): the selection of open source libraries and sys-wrappers is very much in its infancy.
With core dependencies of useful libraries lying untouched, mysteriously broken and littered with `TODO` comments, it really shows.
This 'curio' angle resurfaces when it comes to how forward-thinking wrapper writers might not be, such as `Decoder` state `struct`s for [Opus](https://github.com/SpaceManiac/opus-rs) mysteriously lacking the `Send` Trait despite being, well... `struct`s.

I guess this ties into a 'momentum problem' of sorts: I, as a new contributor to a library, can't and don't want to fork and fix an abandoned dependency, push it as a new project to Cargo and then rally for a migration over to that.
It ends up being easier to bang out code like:

```rust
// Since each audio item needs its own decoder, we need to
// work around the fact that OpusDecoders aint sendable.
struct SendDecoder(OpusDecoder);

impl SendDecoder {
	fn decode_float(...) -> OpusResult<usize> {
		let &mut SendDecoder(ref mut sd) = self;
		sd.decode_float(input, output, fec)
	}
}

unsafe impl Send for SendDecoder {}
```

And this is a problem, I guess, that might well be present in all communities but that the young ecosystem really exacerbates.
It's easier to drop fragile fixes into downstream libraries (which are likely being independently implemented over and over) than it is to fix the libraries themselves.

Another of the language's sore points (that I hope should be resolved in time) is the compiler's performance.
Even with incremental compilation, `rustc` falls so far behind modern C++ compilers that the build process for a project relying on a medium-size library can take dauntingly long.
The suggestion to get around this is `cargo check`, which is useful almost all the time and during a project's infancy.
Yet it *isn't* helpful when you know your syntax or types are correct, and that the problems you're having are purely behavioural.
I'm willing to bet this is the case for most use cases applying data-dependent logic, such as audio processing or game logic.

One of the downsides of the borrow checker (and the ordinarily helpful compiler messages) is that certain access patterns that I imagine are common lead to downright unhelpful error messages.
Suppose you have an `Option<...>` field on a `struct`, and you're aware that it must have a valid value due to another invariant:

```rust
struct Entity {
	id: i64,
	pos: Option<Position>,
	seen: bool,
}

// ...

let mut t = Entity {1, Some(...)};
process(&mut t);

// ...

fn process(t: &mut Entity) {
	// suppose (id < 0) <=> pooled (but unalloc'd Entity)
	if t.id > 0 {
		t.pos.expect("It's a valid entity.").move_by(1, 3);
		// ^^^^^ error: cannot move out of borrowed content

		t.seen = true;
	}
}
```

*Naturally*, the correct course of action is to take `t.as_mut()` first, because the commonly used and advertised `expect` and `unwrap` methods [*will* consume ownership](https://doc.rust-lang.org/std/option/enum.Option.html#method.expect).
This isn't made explicit when you're taught about these functions.
It only really becomes apparent when you're trying to contain a type which isn't trivially copiable.
But the error message is indistinguishable from any other ownership violation, and that makes it so much harder to understand *why* your problem is solved this way (and not, say, with wanton use of `clone`).

## The Bot

The [bot itself](https://github.com/FelixMcFelix/felyne-bot) is very much beloved in the server I designed it for -- it chiefly joins the busiest voice channel in any servers it's a member of, playing ambient noises, music, a constant stream of felyne noises and includes some bonus events.
It's a great source of white noise, and it embodies our server's goofiness, in a way.

As development went on, I realised that I'd have been far better off modelling it as a *probabilistic finite automaton*, but given that most of what I've hacked together gets the job done I'm loath to change it unless inspiration strikes once more.
It'd be an interesting afternoon project, if I get bored.

Apart from that, it's responsible for reporting message deletions in a pre-designated channel, for adminstration purposes.
All I really have to do is cache as many messages as I can on a per-guild basis -- in my case, I implemented a simple ring buffer so I wouldn't have to worry about `Vector` modification or reallocation.

## Final Thoughts

To put it simply, Rust is a language I'd love to learn more of and experiment with further, but I think it pushes a lot of burden onto the programmer in terms of both learning time *and* coding time.
The time penalty is definitely a harsh price to pay at first, though I'm left to imagine that it grows that much easier over time.

I'm torn: when it's at its best, the language really helps you and is genuinely quite enjoyable to work with; at its worst, it's like a never-ending cage match with the compiler.
Whether the difficulty it offsets the cost of hunting down the type of bugs that Rust aims to eliminate is very much an open question, but I'd like to play around with it further.