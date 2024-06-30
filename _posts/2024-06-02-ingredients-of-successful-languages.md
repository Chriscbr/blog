---
title: On the ingredients of successful programming languages
---

Ever since I started programming in my early teens, I've been curious about the design of different programming languages and how they shape how we solve problems.
To that end, I feel extremely fortunate that I've been able to work at a startup where we've taken a shot at building a new programming language named [Wing](https://www.winglang.io/) to make it easier to leverage the modularity and scalability of cloud services -- and the past two years I've worked on it so far have been both immensely challenging and rewarding.

One of the realizations I've been humbled by while working on it is just how difficult it is to bring programming languages off of the ground.
In ["The Economics of Programming Languages"](https://www.youtube.com/watch?v=XZ3w_jec1v8), Evan Czaplicki, creator of Elm describes several of the responsibilities of maintainers of languages, including (but not limited to): compiler engineering, build infrastructure and release management, writing documentation, social media posting, blogging, talk presenting, triaging bugs and feature requests, supporting users and contributors, managing trademarks, as well as doing accounting and taxes for anyone working on the project full time.
There's a lot that could be said on the topic of organizing this effort and how to do it sustainably that I'd probably need to write another post to cover in full.

However, I'd like to make the claim that the success of a programming language is not purely a function of the time and human resources available to its authors.
There are several languages that have seen widespread adoption despite having limited numbers of paid maintainers (like Rust), and there are also many languages that have failed to penetrate the industry more broadly despite having sizable corporations and teams supporting them (like Dart or Flow).
Since improving language adoption is one of the challenges I work on at Wing, I've spent a decent amount of time thinking about what common elements, if any, can be gleaned from general purpose languages that have succeeded at reaching sizable adoption in industry.

To that end, I want to put forward a unifying theory for what ingredients are needed for a programming language to be adopted widely.
It's possible (read: likely) that there are many other factors that also matter, but I think these four make up the highest order bits.
In no particular order:

1. How useful and composable its primitives are for solving problems
2. How easy it makes changing and maintaining code
3. How well it interoperates with the rest of the software world
4. How efficient it is in terms of speed, memory, or other costs

Let's break these down.

### 1. How useful and composable its primitives are for solving problems

Primitives are the fundamental building blocks that a programming language grants its users.
These are typically the values, types, objects, or functions that a language is designed around, and which most everything else (libraries and programs) are built on top of.

The choice of primitives in a language tends to affect the types of programs people write with it, because some primitives are more useful for certain problems than others.
Here are some examples ways people use programming languages, and what primitives or "primitive-like elements" might be more useful for them:

| Problem | Relevant primitives |
|---|---|
| creating machine learning models | tensors, matrix operations, graphs, activation functions |
| running queries on data sets | tables, joins, filtering, aggregation functions, indexing|
| creating art or visualizations | shapes, image processing operations, shaders, color manipulation, animation primitives, vector graphics |
| making games | physics operations, collision detection, sprites, event handling, game loops, audio processing |
| designing user interfaces | inputs, forms, layout operations, event listeners, state management, components |
| building server applications | green threads, HTTP connections, middleware, routing, authentication mechanisms |
| controlling robots | sensors, actuators, kinematics, control algorithms |
| modeling simulations or experiments | random number generators, statistical functions, differential equations, agents |
| performing financial calculations | currency types, arbitrary-precision arithmetic |
| writing networking applications | sockets, protocols, packet manipulation, connection handling, async I/O |

Essentially, a language is better for solving certain kinds of problems if its primitives are better posed towards expressing solutions for those problems.

Even though many programming languages _are_ general purpose, it's not really feasible to solve all problems equally well, because there are inherent tradeoffs between different sets of primitives, and which primitives the language presents as easiest or most natural to use.

If numbers in a language are stored as strings by default, it might be better suited for financial or scientific computing where numeric precision is critical -- but this default would be at the cost of the experience of someone trying to implement fast physics simulations.

If a language presents functions and function composition as the most natural way to organize programs, the language might be very convenient for writing complex data transformations and composing pipelines thereof.
But it might come at the cost of the experience of game developers who wish to model each item and enemy as an object with methods.

Languages can try to include large numbers of primitives to satisfy everyone -- but this too can come at the cost of making it harder to learn the language, and harder to build tooling around it (whether for metaprogrammers or language maintainers).

To illustrate this, we can see how useful programming language-like environments have been developed by curating subsets of existing languages and adding primitive-like library functions to them, as in https://p5js.org/.

### 2. How easy it makes changing and maintaining code

Programming languages are fundamentally designed for humans.
Computer hardware today isn't designed for running Python -- it's designed for millions of low-level ADD and MOV instructions, and programming languages like C, Python, and Rust are just here to make the process of instructing the computer a lot easier for us.

(I'd claim this still holds true in the age of [AI](https://www.cognition.ai/blog/introducing-devin) [programmers](https://github.blog/2024-04-29-github-copilot-workspace/). As long as the software is being written for us, humans are going to be in the loop of ensuring that software works, which means we need the ability to read and debug code).

To that end, modern programming languages include all kinds of design decisions to make programs easier for us to write:

- Lexical scoping makes it easier to  
- Functions make it easier to reason about complex processes than GOTO statements
