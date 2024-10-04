---
title: Unsolved problems in Winglang
---

Wing (or Winglang) is a general-purpose programming language designed to make it easier to build cloud applications.
This post isn't meant to be an introduction to the language, so if you want to learn more about how it came about or what it's like to write code in it, see [this blog post](https://www.winglang.io/blog/2022/11/23/manifesto) or the [Wing docs](https://www.winglang.io/docs).

<!--
When I say cloud applications in this post, I tend to mean applications with a heavy infrastructure component.
Applications like these aren't well described by a single executable, because they depend on many other services, like Redis, S3, EC2, Lambda, Kafka, Cloudflare, etc.

In Wing, instead of defining these services *outside* of your application through a configuration file (like Kubernetes YAML or Terraform HCL), you define them inside your application as special compile-time objects.
Then, you can write business logic that uses those objects, and the compiler will automatically wire it all together.

An example of a program in Wing looks like this:

```js
bring cloud;

let api = new cloud.Api();
let store = new cloud.Bucket();

api.get("/employees", inflight (req) => {
  let var processed = 0;
  for key in store.list("inputs/") {
    let contents = store.getJson(key);
    let cInfo = contents["customerInfo"];
    store.putJson("processed/{key}.json", cInfo);
    processed += 1;
  }
  return {
    status: 200,
    body: Json.stringify(processed)
  };
});
```

-->

<!-- The syntax of Wing is heavily inspired by the likes of TypeScript, Swift, and Rust, but it also introduces a few keywords of its own. -->

<!--
The code example above imports the `cloud` module from the standard library, creates two resources (an API router and an object storage bucket), and specifies a function to run when the "/employees" endpoint is called. That function processes files in the bucket and returns a count of how many items were processed.

The Wing toolchain can compile this program into a set of files that you can deploy to AWS using Terraform/OpenTofu, or it can compile the program into a local simulation.
-->

Today, Wing has been in development for about two years, and is still pretty nascent as I write this.
It has grown to include many of the kinds of features you might expect from a modern language, like control flow, functions, classes, interfaces, enums, optional types, error handling, a standard library, third-party package management, IDE support, and more.
It is also designed to make it more productive for building and maintaining cloud applications: cloud services are first-class citizens, the compiler automatically infers cross-resource permissions, JavaScript code can be brought in and used, and more.

My time spent working on the language at [Wing Cloud](https://wing.cloud/) has involved both lots of testing (building applications with it), and lots of time designing and implementing features to support these technical requirements.
In this post, I wanted to try and explore some of the technical challenges, and explore what I think are some of the biggest unsolved problems in the compiler to date.

> Disclaimer: The design of Winglang has been a joint effort by many people. All of the opinions I'll be sharing are just based on one way I'd imagine the language growing.

## Generics

Generics is the fancy term for code that uses a type as a parameter in some way.
In many languages you might see `List<T>` or `List[T]` used to represent a list of elements of type `T`, where `T` could be anything from a `string` to a `CalicoCat`.

Wing doesn't really have support for generics right now.
There are a few built-in generic types, like `Array`, `Set`, and `Map`, but besides those, your classes and functions must all be created using "concrete" types.

In a general-purpose language, not having generics can make it hard to write many kinds of custom data structures. For example, there's no way to define your own `BinaryTree<T>` type.

But there are also some scenarios where it seems like generics could be particularly useful for cloud programming.
For example, Wing's standard library includes a class named `Function`. `Function` is an abstraction for a serverless function service, like [AWS Lambda](https://aws.amazon.com/lambda/).
When you call a serverless function on one of the big public clouds, it quickly spins up an instance, runs the function code, and stops the instance -- you're only billed for the time your code executed.

Like a "traditional" function, a serverless function can take any input and return any output, so it would be nice to be able to specify the input and output types up front:

```js
bring cloud;

let fn = new cloud.Function<PaymentInfo, Order>(inflight (payment) => {
  payment.validate();
  let order = payment.createOrder();
  return order;
});
```

In the code example above, `PaymentInfo` is the type of the input, and `Order` is the type of the output. By doing so, we can get the benefits of static type checking -- if someone tries to invoke the function with the wrong type, it will error:

```js
api.get("/process-payment", inflight (req) => {
  fn.invoke("123"); // error: "123" is not a PaymentInfo
});
```

To work around Wing not having generics today, `Function` expects `Json` input and output values, and users can use helper functions to convert most structs to-and-from `Json`. This kind of solution requires more boilerplate, but its workable.

Many languages have generics today, so it would be handy to add to the language. But building out support for generics well could involve handling many tricky edge cases.

For example, we ought to have a way to express that the generic type parameters in `Function` should be "serializable" types since invoking a serverless function usually requires making an HTTP request under the hood. Some kinds of values in the language, like functions, are not serializable.

## More robust permission inference

A major feature distinguishing Wing from traditional languages is how it uses static analysis to wire together compile-time infrastructure information and runtime code that uses the infrastructure. This avoids the need for a lot of glue code needed when using other infrastructure-as-code frameworks.
Here's a simple example:

```js
bring cloud;
bring http;

let api = new cloud.Api();
api.get("/greet", inflight () => {
  return { status: 200, body: "Hello, world!" };
});

new cloud.Function(inflight () => {
  // call the api's "/greet" endpoint
  http.get(api.url + "/greet");
});
```

When you are writing this application, you probably haven't deployed it to AWS yet, so you don't know what the API's URL is.
Neither does the compiler.
But nonetheless, we can still compile this program and have the correct URL automatically injected when the app is deployed.

> Note: The way Wing currently does this is the compiler identifies that `api.url` is an unresolved value, and inserts a piece of JavaScript code like `process.env.API_URL` in its place. Then, it adds an environment variable to the serverless function with a matching name, whose value will be resolved by the underlying provisioning engine, usually Terraform or CloudFormation. All of this is done automatically.

If you were to try and do the same thing in [SST](https://sst.dev), you'd need to pass an extra `link: [api]` parameter to the `Function` class, and then in your function's code (in another file) you would have to import the URL through a magic global variable:

```js
import { Resource } from "sst";

console.log(Resource.MyApi.url);
```

The way SST does it in this scenario isn't too bad since we're dealing with simple strings, but as your application grows and changes, this can become more error-prone.

Here's a more involved example:

```js
bring cloud;

class ReplayableQueue {
  queue: cloud.Queue; // a distributed queue
  bucket: cloud.Bucket; // an object store
  counter: cloud.Counter; // an atomic counter
  
  new() {
    this.queue = new cloud.Queue();
    this.bucket = new cloud.Bucket();
    this.counter = new cloud.Counter();
  }
  
  pub inflight push(m: str) {
    let count = this.counter.inc();
    this.queue.push(m);
    this.bucket.put("messages/{count}", m);
  }
  
  pub inflight replay(){
    for i in this.bucket.list() {
      this.queue.push(this.bucket.get(i));
    }
  }
}

let rq = new ReplayableQueue();

let api = new cloud.Api();
api.post("/push-item", inflight (req) => {
  rq.push(req.body ?? "<empty>");
});
```

In the first section, we create a class named `ReplayableQueue` with two methods, `push()` and `replay()`. `push()` adds the message to the queue and persists it to an object store. `replay()` accesses the object storage and pushes all of the items back into the queue as new messages. In the second section, we create an API endpoint that calls the `push()` method of the queue.

<!-- One advantage of using Wing here is that we're able to define a well-encapsulated abstraction. The code for defining the ReplayableQueue can be defined elsewhere in your application, and the choice of whether to persists queue messages using an object store or using an in-memory cache etc. can be made private, and can even be changed in the future. -->

The reason this code works is that the Wing compiler can statically analyze the call graph of the application. In particular, it can deduce that the "/push-item" endpoint handler calls `ReplayableQueue.push()`, which calls `this.counter.inc()` and `this.queue.push()` and `this.bucket.put()`.
That means the corresponding infrastructure needs to have write permissions to the underlying Queue, Bucket, and Counter. The infrastructure must also have access to the physical resource names (for example, the S3 bucket name) for making API requests.
This extra wiring is handled automatically by the Wing compiler.

---

All of this can feel a bit like magic, but today it depends on the fact that we can trace the call graph down to the concrete instances of Queue, Bucket, and Counter.
However, if we tried writing some code that was a bit more general (using a functional style), it wouldn't work in Wing today:

```js
// a function that empties all the objects from a bucket, one by one
let emptyBucket = inflight (bucket: cloud.Bucket) => {
  for key in bucket.list() {
    bucket.delete(key);
  }
};
```

> Note: to the observant commenter, yes, using batch operations would be more efficient here.

Wing raises an error if you try to compile this program because it doesn't currently have a way to model permissions when a resource is a parameter of the function. That means if you tried deploying an app that called this function, it would just fail at runtime.

My hunch is that it should be possible to support this by expanding the static analysis performed today to reason about more complex control flow.
[Data flow analysis](https://en.wikipedia.org/wiki/Data-flow_analysis) is the general term for a set of techniques for tracking how data moves around a program, and it may be possible to apply these techniques to Wing's type inference and permission inference problems.

## Type inference

Type inference is a common feature for many statically typed languages. Type inference means the types of variables are figured out automatically when your program is compiled so that you don't have to add type annotations everywhere in your source code.
The simplest possible example in Wing looks like this:

```js
let color = "vermillion";
```

Here, the variable "color" is implicitly assigned the type `string` based on the right-hand expression.
Without type inference, you would have to type:

```js
let color: string = "vermillion";
```

Type inference also makes it possible to omit the types of parameters in many situations:

```js
let average = (x, y) => {
  return (x + y) / 2;
};

average(4.1, 38);

// ... instead of

let average = (x: num, y: num) => {
  return (x + y) / 2;
};
```

Implementing type inference becomes complex when multiple facts or pieces of information have to be "chained" to form an inference. For example:

```js
let api = new cloud.Api();
let func = inflight (request) => {
  return {
    body: request.body,
    status: 200,
  };
};
api.get("/hello/world", func);
```

When the compiler reads this code, after it scans the code for `func`, it doesn't know what type the parameter `request` should be, so it can't be sure whether accessing `request.body` is allowed.

<!--
> Note: In a dynamically typed language, `request.body` would always work (it just might return a value like "null" if it's invalid) - but in Wing, we'd like to give you an error if you access an invalid property.
-->

But once the compiler reads the last line of code, it can look up the type of `api.get`, including what types of arguments it expects.
It can then infer that `func` should match the type expected by the second parameter of `api.get`.
That allows it to infer that `request` has the type `ApiRequest`.
`ApiRequest` has a field named `body`, which means the compiler can ultimately determine that accessing `request.body` is valid.

Unfortunately, type inferences aren't always simple "chains" like the one above. Sometimes you want to infer a type based on multiple lines of code, like the example below:

```js
let fn = (condition: bool) => {
  if condition {
    return "hey";
  } else {
    return nil;
  }
};
```

Here, we would like to infer that the return type of this function is `str?` (an optional string) since either a string or nil is returned, but this doesn't work in Wing today.

There are well-known algorithms for solving this type of problem, like [Hindley-Milner type inference](https://course.ccs.neu.edu/cs4410sp19/lec_type-inference_notes.html), but sometimes the error information produced by these algorithms can be confusing to users, and in other cases it can [slow down compilation a lot](https://danielchasehooper.com/posts/why-swift-is-slow/), so some care would be needed to implement such an algorithm and make it production-ready.

There is a lot of great reference material about implementing type inference, so I'd like to dive into the topic and try implementing it myself sometime.
(I believe implementing a system is one of the best ways to understand how it works).
[Here's](https://go.dev/blog/type-inference) a post from the Go language team that I haven't read yet but looks promising.

---

Here is one more Wing example that might be interesting to look at:

```js
let fn = (name: str?) => {
  if name == nil {
    return "Hello, world";
  } else {
    return "Hello, {name}";
  }
};
```

If you've used a language like TypeScript, you would probably expect this code to compile.
Today, Wing raises an error message on the third-to-last line saying that `name` could be nil, so it can't be interpolated into the rest of the string.
This doesn't seem right - the if condition explicitly checked that `name` was not nil, so why does the compiler still complain in the else branch?

I'm not sure if this problem would be solved by type inference, but it does feel closely related.
I suspect by using data flow analysis or other techniques, the compiler could assume that any code inside the else-branch is working with `str`, not `str?`.
In TypeScript, this feature is called [type narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html).

It's possible to work around this limitation today by "unwrapping" the contents of `name` or using the [`if let`](https://www.winglang.io/docs/api/language/optionality) syntax, but most folks that have tried Wing expect this kind of code to work on their first try, so supporting it seems like the right thing to do.

## TypeScript language interop

As powerful as any one programming language is, we have to acknowledge that almost all software is [built on top of the shoulders of giants](https://xkcd.com/2347/), so writing every line of code from scratch is rarely practical for most projects.
For that reason, most languages must have ways to reuse existing code.

The most common way to support this is with something called FFI, or [foreign function interfaces](https://en.wikipedia.org/wiki/Foreign_function_interface).
With FFI, you declare in your source code what functions you are using from the other language, and then the compiler will wire the code together so it can be used.
Here's what FFI looks like in Wing today:

```js
class BookShop {
  extern "./book-shop.ts" static slugifyBook(title: str): str;
}

BookShop.slugifyBook("Romeo and Juliet");
```

Since Wing compiles source code into JavaScript, it made natural sense to support FFI with JavaScript and TypeScript.
However, there are a few caveats to this approach.

First, if you need to use more than a handful of functions from TypeScript, it gets pretty cumbersome to write all these definitions by hand.
It would be useful to have a tool that automatically generates these bindings based on type information from `.d.ts` files.

Moreover, the type systems of Wing and TypeScript don't fully match up.
While some types behave the same between languages (primitive types, classes, interfaces), there are also many types in TypeScript that can't be easily expressed in Wing, like `string | number`, or `Person & Serializable & Loggable`.
Wing's type system also has some features that don't map cleanly to TypeScript, like the distinction between *interfaces* and *structs* (Wing structs don't have behavior associated with them, while interfaces do).
My hunch is that this distinction is a holdover from the early days of the language and that it may be possible to remove it and simplify the language.

Ironing out these details might require large overhauls, but I think it would pay off greatly in making the language more approachable.

## Bonus: Formatting

Learning a new language always has a curve to it, so the more that a toolchain can help you out and get up to speed faster, the better.
Wing has a [language server](https://en.wikipedia.org/wiki/Language_Server_Protocol) which provides a lot of features like showing errors in your IDE and method auto-completions, but one feature it's missing is code formatting.

Wing doesn't have a code formatter yet, but it would be great to have one.
Code formatters smooth out the experience of updating and maintaining code, especially when you are working with a team.
Since the language is still actively being developed, it feels like the best thing to do might be to implement it as part of the existing compiler, so that code for parsing and formatting can be updated in a single place.
