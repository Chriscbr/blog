---
title: "Multi-language software projects suck to work on"
---

They can be done, but it's an uphill battle.

When I talk about "multi-language software projects," I mean projects where core business logic is split across multiple languages - the real meat-and-potatoes stuff with functions, branching logic, and data processing.

To date, I've worked on projects with both TypeScript and Python, Python and Go, and Rust and TypeScript, but you can find a lot more combinations in the wild.
Linux, for example, has recently started adopting Rust into its historically C-based codebase under the [Rust for Linux](https://en.wikipedia.org/wiki/Rust_for_Linux) project, to mixed success.

I think the real question you need to ask yourself is, if your software project has business logic in multiple languages, how does the code talk to each other?
And how much does someone writing code in one language need to know about the code written in the other language?

If all the talking between languages happens through a simple interface, it's manageable.
Take CRUD apps: they often use JavaScript for the frontend (browsers demand it) and something else for the backend - Java, Python, Go, whatever.
When data can only flow through a few well-defined endpoints using JSON, friction stays low.
Moreover, the separation of concerns is strong enough that you can specialize in one half of the stack without needing deep knowledge of the other half.

But when languages interact through complex interfaces, you tend to need deep knowledge of both parts of the codebase just to make simple changes.
For example, once [FFI](https://en.wikipedia.org/wiki/Foreign_function_interface) is involved, you need to be aware of how types are laid out in memory and how each language manages its memory.
In an old enough codebase, business logic might be duplicated or inconsistently implemented across languages.
You'll also need to master multiple debugging tools and environments.
In short, this means the project will suck to work on.

<br>
<center><b>·</b></center>
<br>

Hold on, you say.
Learning new programming languages isn't really that hard.
What's the big deal?

There's a surface level truth to this claim.
Learning your first programming language requires learning several concepts (variables, for loops, strings, functions...) -- but once you know that one language well, you have enough of background to learn a second language much quicker.
Your third and fourth languages will be even easier to learn.

However, the bigger the project you're working on, the more code it will have, and likely the knowledge you're going to need about the languages it's written in.
Every language has boundless trivia about standard library features, memory layouts of data structures, and oh so many conventions and idioms.
If features and bug fixes regularly require code changes in multiple languages, this means more prerequisite knowledge for the developer.
This limits how many people can work on the project and how quickly new developers can get up to speed.

<br>
<center><b>·</b></center>
<br>

If you're thinking about splitting business logic across multiple languages, ask yourself: is it worth the cost?

The best software projects are ones where developers can move fast.
If your project forces them to juggle multiple languages just to get things done, you're slowing them down.
Keep it simple.
