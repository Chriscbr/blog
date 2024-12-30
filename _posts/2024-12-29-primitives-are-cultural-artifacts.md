---
title: "Primitives are cultural artifacts"
---

<center>

<img src="/blog/assets/images/building-blocks.png" width="450">

</center>
<br>

One of my interests in programming is making software for other software engineers.
Software development requires building up big things from lots of little things, so a concept that's on my mind a lot is the idea of the "primitive".
The word primitive comes from the Latin *prīmitīvus*, which means "first or earliest of its kind."
But the definition I like to use is that a primitive is a building block.
Like LEGOs, primitives are pieces designed to be combined together in order to form larger structures.

Because the word "primitive" has an academic, highfalutin sound to it, I think folks working on software like using the term more than other professionals, but the idea extends beyond software.
Take chemistry, for example.
The periodic table contains all of the elements that make up matter in the universe: chemical elements are the primitives of *matter*.[^1]
Or take cooking: inside a kitchen, a chef stores all of the ingredients they need, from salt and pepper to flour and sugar and beyond.
All dishes are made of one or more ingredients, composed together by cooking or mixing them one way or another: ingredients are the primitives of *cooking*.

Here's an open question.
For a given domain (like cooking, chemistry, or programming), is there an optimal set of primitives?
Is there a "correct" or "best" set of primitives?

## Picking the best primitives

If we just look to the chemistry example, it seems like the answer to our previous question should be yes.
We know there's a fixed set of elements in the periodic table -- there can't be elements with seven and a half protons.[^2]
It seems like there's a single, correct set of primitives here.

But if we turn to cooking instead, it's not clear what set of primitives is best.
Salt and pepper seem like reasonable choices for primitive ingredients... but they're just used to flavor food.
Why would these be primitives and not other spices?

Is cheese a primitive?
Cheese is made from milk and bacteria cultures - those raw ingredients could certainly be primitives.
But most chefs won't go to the effort of making their own cheese because of the time it takes, so it might be worthwhile to treat store-bought cheese as a primitive too.
It seems like there's a lot of flexibility here.

Finally, if we look to programming, trying to find the best set of primitives is even harder.
[Assembly language](https://en.wikipedia.org/wiki/Assembly_language) tends to give a programmer the most direct and flexible control over a computer's hardware, making it a good candidate for a primitive.
But writing assembly code by hand is too tiresome and error prone, so almost all programmers use one or more higher-level languages instead.

There are also hundreds of languages (both assembly and higher-level languages), and each language has its own set of primitives inside it.
For example in Python, all integers are represented with an `int` type, while in Rust, there are over a dozen ways to represent an integer.

Unfortunately, it's starting to look like there's no best set of primitives.
What you *can* have are sets of primitives that are more or less useful.
That raises the question...

## What makes a good set of primitives?

Here are some aspects shared by great primitives in both cooking and programming:

* **The primitives are flexible and multi-purpose.**
    * An ingredient like milk can be used as a liquid base for drinks, it can be turned into a béchamel, it can be added to soups for creaminess, it can be frozen for ice creams, etc.
    * A software primitive like an array can be used for sorting records in a dataset, or for buffering requests in memory, or for storing pixel data for an image, etc.
* **The primitives don't introduce a lot of waste.**
    * When you use milk as an ingredient, you might toss or recycle its container when it's empty, and that's it. Likewise when you use a fruit as an ingredient, usually everything but the peel can be used.
    * When you use an array in programming, there may be some extra information stored alongside it (like the size of the array), but the overhead beyond the array's contents is minimal. An array is also performant - it always guarantees constant-time random access.
* **The primitives aren't complete, pre-made artifacts.**
    * A frozen dinner can be ok to eat, but it's not a useful primitive for a chef for creating new dishes.
    * Similarly, a Wordpress website can be ok for hosting your blog, but it's not a useful primitive for a programmer for solving many other problems.
* **The primitives are accessible.**
    * Most cooking ingredients can be readily used and combined. Sometimes they require thawing or peeling, but that's it. If an ingredient is hard to prepare, it's probably not needed for most dishes.[^3]
    * Most programming languages make their primitives built-in, meaning you just have to type one or two keywords to invoke them. Primitives also are documented to make them easier to adopt.
* **The primitives are not substitutable for each another.**
    * When any ingredient is replaced in a recipe, there's always a tradeoff because of its different chemical properties and flavor profile. Whole milk, heavy cream, and oat milk can all be used in coffee, but they will taste different. It makes more sense to keep different kinds of dairy products in a kitchen than to keep different brands of whole milk.
    * Primitives like numbers and lists can be represented all kinds of ways on a computer, but they typically have different memory representations which may incur different storage or performance costs. When software primitives aren't distinctive enough, it becomes [hard to distinguish between them](https://x.com/forrestbrazeal/status/1400639759215640577).

## TL;DR

Ultimately, a set of primitives isn't an a mathematical ideal. Every set of primitives is shaped by what problems its users are solving.

Even when we think of primitives from an maths perspective, like as a set of axioms that define a complete system, we tend to find that there are multiple ways to define a system.
For example, both Turing machines and lambda calculus are complete systems for defining computation[^4], but they're different.

The ideal set of primitives for a chef making restaurant meals can be different from the set of primitives of a parent making school lunches for their kids. Likewise, the set of ideal primitives for programmers working on backend software, mobile apps, websites, and games can all be different.
All sets of primitives are shaped by the technology that's available and the needs of the people using them.

Designing a set of primitives without a specific area of focus is unproductive because primitives that try to appeal to everyone tend to be the best for no one.

---

[^1]: There are other kinds of "things" in the universe that aren't matter, like light, too.

[^2]: For any whole number of protons, you can theory-craft a new element, so the set of elements could be infinite -- but the majority of them would be unstable or impractical to produce.

[^3]: Some ingredients like fish might be tricky to prepare if you're trying to extract every edible part, but even then, obtaining the main fillet portion usually only requires a few slices.

[^4]: By complete, I just mean that they're equivalent in power. A Turing machine can simulate a lambda calculus program, and a lambda calculus program can simulate a Turing machine program.
