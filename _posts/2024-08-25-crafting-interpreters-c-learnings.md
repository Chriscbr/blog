---
title: Tidbits on C after reading Crafting Interpreters
---

I recently finished reading through [Crafting Interpreters](https://craftinginterpreters.com/) - a book I'd have to vouch for as the best introduction to writing interpreters or compilers.

There's a lot that can be said about the design and implementation of programming languages and their toolchains, especially as today's developers demand more features like automatic documentation, package management, IDE extensions, and so on.
To that end, how people design and implement languages varies widely (some take a more theoretical or academic approach, and others dive deep into optimizations and implementation tradeoffs).
Parsing, data flow analysis, type checking, code generation, code optimization -- all of these areas can be studied as fields of their own.

Crafting Interpreters offers an excellent middle ground that spends time introducing many of the key ideas and algorithms, while keeping it grounded to keep the focus on the implementation of building a real interpreter.
The code in the book is fairly clean, and the author even opines at some points on good software development practices.
Likewise, the language implemented in the book, "Lox", is incredibly pragmatic in its design (it's similar to Python or Java), reflecting the design choices that people expect from some of the most common general purpose languages.

Anyways, that's enough praise about the book for now.
While my main goal reading the book was to learn about compilers, by copying every line of the book by hand into my editor, I ended up learning quite a bit about C that I don't think I had known before, so I thought I'd share some of these takeaways.
Some of these were explained in the book, while others were the result of my own Googling.

- The `static` keyword can serve multiple purposes in C. One is that it can be used to specify a function is scoped to the file it's in - sort of like a "private" function. Another purposes is that a variable can be labeled `static` to make its value shared between invocations. This is a pretty interesting feature that not a lot of other languages have.
- You can define functions in header files, but only if they're `static inline`. To be honest I'm still not familiar with the tradeoffs between defining a function in the header file vs. the main implementation file.
- C supports a form of "inheritance" through structs, informally called *struct inheritance*. The idea is you define a collection of child structs that all have the same first N fields, as well as a parent struct which just has those N fields, and by doing so, it's technically safe to cast from the child struct to the parent struct. 
- Struct members which are arrays (not just pointers) almost always must have a fixed length. This makes sense when you think about it, since otherwise you wouldn't be able to calculate the size of the struct. The only edge case is something called a [flexible array member](https://en.wikipedia.org/wiki/Flexible_array_member) which has to be the last field in a struct. If you're doing this, you have to make sure you're allocating the correct amount of memory for the struct and any array elements you want to store in that member.
- Many functions from the C standard library have super short names because early C linkers only treated the first six characters of external identifiers as meaningful.
- C enums are generally specified to be at least large enough to hold an int. So if you want to hold something small and compact (esp. within a struct), it's better to use a different type, or to cast enum values to/from `char` values, etc.
- C supports functions with variable arguments; just the way you do it is a little bit weird. (You have to create a local variable of type `va_list`, then call some functions named `va_start` and `va_end`, and then do some other stuff).
- There are many different kinds of "NaN" representations possible in the way C and other languages model floating point numbers. "IEEE 754 divides \[NaNs] into two categories. Values where the highest mantissa bit is 0 are called signalling NaNs, and the others are quiet NaNs."

A lot of these notes I took a month ago so unfortunately I don't have the sources for all of them saved - please do your own due diligence if you plan to use these features of C.

One more observation.
Throughout the second half of the book, there were comments making a distinction between whether a pointer in a struct was "owning" a piece of data (i.e. if it's responsible for deallocating the memory it points to when it's done), or whether it was simply holding a reference that is actually owned by a struct elsewhere.
In Rust, this information would be made explicit: an owned pointer to a type `T` is `Box<T>`, while an unowned pointer to a type `T` would be `&T` or `&mut T`.

These comments in the book were illuminating to me since I felt like having to mentally track this ownership information while working with the C code gave me empathy for how Rust aims to solve some of these problems.
