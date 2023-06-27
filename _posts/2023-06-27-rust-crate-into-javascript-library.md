---
title: How to turn a Rust crate into a JavaScript library
---

Hi there - my name's Chris.
I'm an avid software engineer that enjoys building things.

At work, one thing I've been helping build is a compiler.
(The compiler is for a language named Wing[^wing] - but there's no time to dwell on that in this post.)

[^wing]: https://www.winglang.io/

## Motivation: quality error messages

For building the compiler, we decided one design tenet would be to have nice error messages.
Nice error messages don't just make a tool less frustrating, but they also teach the user how to use the tool.

One way to make error messages nicer is when they show you _where_ your error is.
Rust has a number of crates [^crates] for displaying nice error messages:

[^crates]: Crates are the name for libraries in the Rust ecosystem.

```
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:5:28
  |
2 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = s1;
  |              -- value moved here
4 |
5 |     println!("{}, world!", s1);
  |                            ^^ value borrowed here after move
  |

For more information about this error, try `rustc --explain E0382`.
error: could not compile `ownership` due to previous error
```

As a byproduct of the Rust community's [appreciation of great error messages], Rust has several crates for generating well-formatted error diagnostics, like [miette], [codespan-reporting], and [ariadne].

[appreciation of great error messages]: https://alexwlchan.net/2021/rust-errors/
[miette]: https://crates.io/crates/miette
[codespan-reporting]: https://crates.io/crates/codespan-reporting
[ariadne]: https://crates.io/crates/ariadne

For our compiler, we decided to make our command-line interface (CLI) responsible for formatting error messages - however, the CLI was written in TypeScript (aka JavaScript with types).
It would be great to reuse the diagnostic formatting tools from Rust.
What's one to do?

_WebAssembly to the rescue!_

WebAssembly is a binary instruction format that can be run on many platforms, including browsers and most JavaScript runtimes; and Rust natively supports compiling most programs to WebAssembly.

This means if we can reuse Rust libraries in our CLI if we compile them into WebAssembly first.

**Goal:** Create a JavaScript library (npm package) that lets us use all of the capabilities of a Rust library.

## Scaffolding the library


