+++ 
title = "Initial Thoughts on Rust"
date = 2018-02-06T12:07:53Z
description = "A language I want to love."
image = "blog/felynes.png"
tags = ["Rust", "Discord", "Monster Hunter"]
categories = ["Development", "Programming Languages"]
draft = true
+++

Having been bereft of a way to annoy my fellow Discord users (and itching to learn a new language), I've found myself experimenting with [Rust](https://www.rust-lang.org).
It's been fun and tiring writing [Felyne bot](https://github.com/FelixMcFelix/felyne-bot) in a new language, and I've gotten some more language experience by helping develop [Serenity](https://github.com/zeyla/serenity).

<!--more-->

---

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

---

I feel I should start with the good parts of the language (although, I must disclose I haven't tried to toy with macros).

Bringing the program model back down to *structs* and *Traits* (like C, with the kind of generic type reasoning we all love) is very close to the level of abstraction I like to work with.
It's a move away from a certain Object-Oriented dogma that's been floating around for a long time now, but keeping the higher-kinded reasoning (and quality of life) that exists in most languages today.
It's very much in-line with the whole 'zero-cost abstraction' philosophy the devs are going for, and I think it's more than enough to encode fairly rich relationships.

Compiler error messages are a god-send.
Not that I like having my work picked apart, but the clarity and pointedness of the messages themselves is so far above and beyond what other compiled languages really have to offer.
Too often are the days when a relatively simple error in C++ (for instance) leads to pages upon pages of useless information spewing forth from some template code violation.
It's *not* something I want to be spending the time to unpack, and Rust's compiler generally knows how to help you along the path to a working program---the borrow checker is notorious, after all!

---

?? Issues? Infancy/abandomnent of libraries and sys-wrappers, language wants to fight you at every turn... Build times. Option access on structs, and the compiler not giving helpful messages.

---

Future work? Slight glitches, proper FSM mmodelling...