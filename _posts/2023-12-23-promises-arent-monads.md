---
title: Why JavaScript promises aren't technically monads
---

While I was chomping down on my lunch the other day, I was web browsing when I stumbled on [this tweet](https://twitter.com/paul_snively/status/1735225290689851556).

> *@paul_snively*
>
> > *@devagrawal09*
> >
> > Javascript devs: "I don't know what a monad is"
> >
> > Also Javascript devs: "Here's a blog post explaining everything about Promises"
>
> Promises are, famously, NOT monads, resulting from <https://github.com/promises-aplus/promises-spec/issues/94#issuecomment-16176966>, and leading to <https://github.com/fantasyland/fantasy-land>.

Huh?

Promises aren't monads? That's a bit weird, I thought to myself.

Granted, I know fully well that I'm not fluent in the study of monads and monoids and endofunctors and other fancy math concepts from [category theory](https://en.wikipedia.org/wiki/Category_theory).

But, the last time I tried understanding what a monad was, it seemed like... it was some kind of container for other values that you can compose and apply functions to.

I also remembered being told that [array](https://en.wikipedia.org/wiki/Array_(data_type)) and [option/maybe](https://en.wikipedia.org/wiki/Option_type) types are monads.
They contain other values, and you can apply functions to them.

Don't JavaScript promises sort of fit that description?

Unfortunately, it was around that time when I finished my lunch, so I decided to leave the tweet in my collection of open Chrome tabs for later investigation.

Alas, today I stumbled on the tweet again, so I figured it's time to crack this mystery once and for all.

----

First of all, the links shared by `@paul_snively` cover some interesting history from the JavaScript community.
Without trying to misrepresent either side, it seems like functional and imperative programming factions have battled it out in the warzone of a GitHub issue that now spans over 250 comments.
It appears a lot of blood and tears were shed.

But the actual discussion isn't filled with too many educational explanations of monads.
So to get to the truth of my original question, it seems we ought to go back to the question of what is a monad.

Now, the first rule of understanding monads is that once you understand them well, you lose all capability to explain what they are.
At least that's what I've been told.
I do want to understand monads, so trying to explain them might jeopardize this situation I'm in, writing this blog.

So instead of making any serious attempt to describe monads conceptually, let's just dig into some of the properties of monads and see if we can get by.
There are two Wikipedia articles on monads, [Monad (category theory)](https://en.wikipedia.org/wiki/Monad_(category_theory)) and [Monad (functional programming)](https://en.wikipedia.org/wiki/Monad_(functional_programming)).
For now, we can just forget math exists and focus on the second article.
I'll try to summarize the formal definition it gives here, complementing it with some [TypeScript](https://www.typescriptlang.org/) code to put it in context:[^1]

[^1]: If you haven't given TypeScript a whirl before, it's definitely worth a shot. Even though it doesn't have the most sound type system, it has a lot of interesting features that make it possible to express a very wide range of APIs. I recommend the [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html) as a good starting point.

For any monad:

1. There should exist a function `unit` which "embeds" and object `x` into the monad.
2. There should exist a function `bind` that unwraps the monad (to get the object inside), and apply a new function to turn it into a new monad.

In TypeScript, the function would look something like this:

```ts
function unit<T>(x: T): Monad<T> { ... }
function bind<T, U>(x: Monad<T>, f: (x: T) => Monad<U>): Monad<U> { ... }
```

(I think I've scared about half of the readers with these higher-order function signatures.
But don't fear! You can step through most of the examples with pencil and paper, or with the help of a JavaScript console if you ever need to.)

We can see from the types that `unit` takes some value (typed as `T`) and puts it inside of the monad (typed as `Monad<T>`). Meanwhile, `bind` takes an existing monad value (like `Monad<T>`) and uses a function to transform it to another monad value (like `Monad<U>`).

I'll highlight two details worth noting:
- Both functions are generic. This illustrates how composable monads are --
you can put anything inside them, without them having a care in the world. `U` could be the same as `T`, or it could be different - they're just like unknown variables in math.
- If `f` in the second argument of `bind` had the signature `(x: Monad<T>) => Monad<U>`, then the implementation of `bind` could just be `return f(x)`.
(It's worth taking a second to verify that).
But since the signature is `(x: T) => Monad<U>`, the actual implementation has to depend on the monad.

To bring this discussion to reality, let's write these functions for the JavaScript `Array` type (which I'm _pretty_ sure is still a monad).

```ts
function unit<T>(x: T): Array<T> {
    return [x];
}

function bind<T, U>(array: Array<T>, transform: (x: T) => Array<U>): Array<U> {
    let result: Array<U> = [];
    for (const item of array) {
        result = result.concat(transform(item));
    }
    return result;
}

// example usage
const nums = [1, 2, 3];
const doubleAndWrap = (x: number) => unit(x * 2);

const result = bind(nums, doubleAndWrap);
console.log(result); // should output [2, 4, 6]
```

Feel free to pause and grok how each function works.

If you're like me, you've probably used Python's `map()` function or JavaScript's `Array.map` function many times.
The `bind` function above _seems_ like it's doing something similar, but it's actually a little more powerful.
Like `map`, it takes in an array and a function, but for each value in the original array, it can produce **any number of values, including zero**.

This means it's possible to implement both `map()` and `filter()` using `bind()`:

```ts
function map<T, U>(array: Array<T>, transform: (x: T) => U): Array<U> {
    return bind(array, (x: T) => unit(transform(x)));
}

function filter<T>(array: Array<T>, predicate: (x: T) => boolean): Array<T> {
    return bind(array, (x: T) => predicate(x) ? unit(x) : []);
}

// example usage
const mapped = map(nums, (x) => x * 3);
console.log(mapped); // should output [3, 6, 9]

const filtered = filter(nums, (x) => x % 2 === 0);
console.log(filtered); // should output [2]
```

Cool!
It goes to show that `unit` and `bind` can let you express a wide variety of computations with monads.

So assuming promises are monads, then we should be able to implement `unit` and `bind` functions for them as well, right? Indeed we can:

```ts
function unit<T>(x: T): Promise<T> {
    return Promise.resolve(x);
}

function bind<T, U>(promise: Promise<T>, transform: (x: T) => Promise<U>): Promise<U> {
    return promise.then(transform);
}
```

That wasn't too bad!
`Promise.resolve` lets you easily create a promise value of any type, and `promise.then` will chain an existing promise with a function into a new promise.

Indeed, TypeScript type checks this correctly and we've satisfied the definitions.
It looks like we can call it a day here.

----

<br>

<center>~</center>

<br>

----

...oh right! There's a second half to the definition on Wikipedia.
Let's look a little closer.

Besides defining the functions `unit` and `bind`, it says we also need to show that those functions satisfy a number of _laws_.
They're not laws of physics per se as much as they are invariants, like the invariant that adding two positive numbers always gives you back a positive number, or subtracting a number from itself always takes you back to 0.

There are three laws they need to satisfy:

1. **Left identity:** `bind(unit(x), f)` is equivalent to `f(x)`
2. **Right identity:** `bind(m, unit)` is equivalent to `m`
3. **Associativity:** `bind(bind(m, f), g)` is equivalent to `bind(m, x => bind(f(x), g))`

I've tried to present them in a simplified form here, but they're still a bit of a mouthful.

We could try and prove these laws rigorously, but bear in mind we're working with JavaScript, a runtime known for oddities like `[] + []` producing `""`, and `0 == false`.[^2]

[^2]: See <https://www.destroyallsoftware.com/talks/wat> and <https://github.com/denysdovhan/wtfjs>

Perhaps we can first build some understanding by validating examples of these laws with the `Array` monad. Let's start with the **left identity**:

```ts
const x = 5;
const f = (n: number) => [n * 2];

console.log(f(x));             // [10]
console.log(bind(unit(x), f)); // bind([5], n => [n * 2]) = [10]
```

Given the `f` and `x` above:
- `f(x)` yields `[10]` through direct application.
- `unit(x)` yields `[5]`, and `bind([5], f)` will apply `f` to each element and concatenate the resulting arrays together, also yielding `[10]`.

The **right identity** is a little easier.

```ts
const m = [5];

console.log(bind(m, unit)); // bind([5], unit) = [5]
console.log(m);             // [5]
```

We can see that plugging a monadic value like `[5]` into `bind` with `unit` as the "transform" function gives us back the original monadic value, since `bind` does the work of unwrapping the array elements, and `unit` puts each element back into a new array.

For an exercise, take a moment to see that this also holds when the array has multiple elements.

Lastly, there's the **associative identity**.
Intuitively we think of associativity as letting us re-group the order operations are applied (but not necssarily the order of arguments), like how `(a + b) + c` is the same as `a + (b + c)`.
It's the same here, just a little more complicated by all of the monad-ness.

```ts
const m = [5];
const f = (n: number) => [n + 1];
const g = (n: number) => [n * 2];

console.log(bind(bind(m, f), g));
// bind(bind([5], n => [n + 1]), n => [n * 2])
// = bind([6], n => [n * 2])
// = [12]

console.log(bind(m, x => bind(f(x), g)));
// bind([5], x => bind([x + 1], n => [n * 2]))
// = bind([5], x => [(x + 1) * 2])
// = [12]
```

In the first expression, `bind(bind(m, f), g)`, we take a monadic value `[5]`, bind it with `f` to get `[6]`, and then bind the result with `g` to get `[12]`.

But in the second expression, `bind(m, x => bind(f(x), g))`, we first compose `f` and `g` together using `bind` to get a new function that behaves the same as `x => [(x + 1) * 2]`, and _then_ we bind `[5]` with that function to get `[12]`.

Wow! Monads are doing a lot of work.

We haven't proved anything rigorously here.
But if you can play around with the code, I think you'll find that whether you use empty arrays, arrays with multiple values, or even arrays containing other arrays (and likewise, all kinds of functions you want to pass to `bind`), the laws should still hold true.

----

Now what about with promises? How monad-y are they really?

Let's look at this left identity law again.
It says that `bind(unit(x), f)` should be equivalent to `f(x)`.

```ts
const x = 5;
const f = (n: number) => Promise.resolve(n * 2);

console.log(f(x));
// Promise.resolve(10)

console.log(bind(unit(x), f));
// bind(Promise.resolve(5), n => Promise.resolve(n * 2))
// = Promise.resolve(5).then(n => Promise.resolve(n * 2))
// = Promise.resolve(10)
```

This doesn't seem too suspicious.
With the values of `f` and `x` we chose above, `f(x)` produces a promise with 10 inside.
Likewise, if we use `unit` to wrap 5 in a promise, and then use `bind` to call `then`, then we get a promise of 10 as well.

Unfortunately, not all values work out nicely like this.
Suppose instead of plain numbers, we were storing promises inside of the promises?
Let's see what happens, choosing new values for `x` and `f`:

```ts
const x = Promise.resolve(5);
const f = (p: Promise<number>) => p.then(n => n * 2);

console.log(f(x));
// Promise.resolve(5).then(n => n * 2)
// = Promise.resolve(10)

console.log(bind(unit(x), f));
// bind(Promise.resolve(Promise.resolve(5)), f)
// = bind(Promise.resolve(5), f)
// = 5.then(n => n * 2)
// 💥 (error: p.then is not a function)
```

What happened?

It looks like in the in the second expression, the nested promise value got implicitly unwrapped.

It turns out this a part of the [promise specification](https://promisesaplus.com/) in JavaScript.
MDN explains that both `Promise.resolve` and `Promise.then` "[flatten] nested layers of promise-like objects (e.g. a promise that fulfills to a promise that fulfills to something) into a single layer — a promise that fulfills to a non-thenable value."[^3]

[^3]: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/resolve>

The reason for this might not be obvious until you look at some application code.
[ExploringJS](https://exploringjs.com/deep-js/ch_implementing-promises.html) gives an example illustrating how if a function returns a returns a promise, like `asyncFunction2` in line (A) below:

```js
asyncFunc1()
.then((result1) => {
  assert.equal(result1, 'Result of asyncFunc1()');
  return asyncFunc2(); // (A)
})
.then((result2Promise) => {
  result2Promise
  .then((result2) => { // (B)
    assert.equal(
      result2, 'Result of asyncFunc2()');
  });
});
```

Then without a flattening process, you'd need to write extra code to unwrap it, as shown in line (B).

With the flattening process, the example above simplifies down to:

```js
asyncFunc1()
.then((result1) => {
  assert.equal(result1, 'Result of asyncFunc1()');
  return asyncFunc2(); // (A)
})
.then((result2) => {
  // result2 is the fulfillment value, not the Promise
  assert.equal(
    result2, 'Result of asyncFunc2()');
});
```

There may be some API benefits here, but unfortunately for this journey, it means promises aren't monads (and there isn't a simple fix for it).

----

Anyways - I hope that was a fun ride through monad properties and JavaScript promises!

I didn't get around to analyzing the other monad laws with promises, but if you take a look you be able to find similar holes in the right identity and associative laws caused by promise flattening.

Shout out to these articles that also do a nice job covering the subtle nature of JavaScript promises:
- <https://www.siawyoung.com/promises-are-almost-monads/>
- <https://buzzdecafe.github.io/2018/04/10/no-promises-are-not-monads>

While it's a bummer that JavaScript promises aren't real monads, they're "fulfilling" their purpose in the ecosystem fairly well, so I wouldn't "reject" them just yet. :-)
