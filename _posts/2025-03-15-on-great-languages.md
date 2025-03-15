---
title: "On great programming languages"
---

A quality shared by great programming languages is that they make you more effective at writing useful code.

To look at it the opposite way, many languages may have interesting ideas motivating them, or have a variety of ways to make you feel more productive -- but I don't think these qualities are what make great programming languages great.

For example, a programming language could have much more beautiful, ergonomic syntax.
Consider how Kotlin makes writing JVM code much more palatable to write.
But this alone isn't a great reason to port your Java code or start writing everything in a new language.
As much as Java may seem dated, recent Java editions have started adding more syntax sugar to make the language feel more modern.
Syntax *is* important!
But syntax does not make a language great.

For another example, a programming language could make you more productive by generating swaths of code for you.
At some point in a programmer's career, one may begin to notice the repetitiveness of code for marshalling data between formats, parsing files, translating requests into SQL queries, and how undifferentiated it can be if you're writing it all by hand.
Why not have a compiler do these things?
Alas, generating code is important!
But it does not make a language great.

As a final example, a programming language could provide special development features -- fast language servers and debuggers, comprehensive documentation generators, battle-tested standard libraries, a built-in package manager, and more.
How could you build software projects today without these things?
All of these are important parts of a programming language's ecosystem.
But they do not make a language great.

<br>
<center><b>·</b></center>
<br>

So that was a bit about what *doesn't* make a great programming language - now what *does*?

When I say "a great programming language makes you more effective at writing useful code," I think there's two parts to focus on.

First, the language's design is catered towards the "useful code" -- which is just my catch-all for code that's providing the most value in a software project.
This is the code the most unique to your project, which is usually algorithmic, or implementing some kind of domain-related behavior.
This is code that you probably care the most about how it handles different edge cases, or the performance of it, or the portability of it, or how security it is, etc.
If this is the most important code in a project, then your programming language needs to help you most with that code (and the qualities of it you're interested in).

Secondly, the language helps you at being effective.
This can take many forms.

* It could be giving you a way to express relationships or processes that you couldn't before.
For example, many languages support concurrency primitives, but I think Go shines at the way its goroutine and channel primitives allow you to parallelize logic both easily and with a high degree of control, and how the rest of the standard library is designed around it.
Haskell is another example of a language that shines in its expressivity, showing how certain problem and solutions can be modeled more easily using lazy evaluation and algebraic data types.
* It could be giving you finer control over how the code is executed by computer hardware, so you can squeeze out more performance.
Low level languages like Zig offer a great deal of ways to partition data in memory, and often discourage unilateral implementations of common data structures (like strings or vectors), instead encouraging you to design and fine tune your implementation so that you can get the maximum possible performance for your particular application. 
* It could also mean helping you recognize logical errors that could otherwise have cost you days or weeks of debugging.
For example, Rust's borrow checking system is uniquely well posed to identifying memory bugs.

I think a natural side effect of these factors is that when you begin learning a great programming language, and reading code in that language, you learn and grow more than you expect.
You're not simply memorizing commands - you start discovering how different ingredients can be combined in new and more effective ways.
You discover how having certain grammatical patterns or phrases in your vocabulary not only allows you to express ideas more concisely, but also allows you to juggle more ideas in your head at once.
For lower-level languages, you may find yourself learning more details about how a computer works than what you previously knew, and having more [mechanical empathy](https://martinfowler.com/articles/lmax.html#QueuesAndTheirLackOfMechanicalSympathy).

As a concrete example of this, I was recently reading through Peter Norvig's essay [Solving Every Sudoku Puzzle](https://norvig.com/sudoku.html), which walks through his design of a sudoku solver in Python.
There are dozens of essays criticizing the design of Python (and even I'll admit it's not my starting choice for all projects), and those with a traditional computer science education would be unsurprised by Norvig's general solution approach of using backtracking search.
But the way the code is expressed feels uniquely inventive as someone that hasn't written Python in a while.
And I think the essay illustrated to me that for Norvig, Python *is* a great language because it allowed him to be incredibly effective at solving a particular kind of problem.
