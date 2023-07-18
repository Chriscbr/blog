---
title: "On the salient features of Winglang"
---

In the ever-evolving realm of programming languages, where academia and industry have continuously pushed the boundaries of software design, a new player has emerged: [Winglang](https://www.winglang.io/).
As a programming languages enthusiast and self-proclaimed nerd who also experienced some of the pain of building apps on AWS, I was drawn to the challenge of building a language that embraced the unique challenges and possibilities of cloud computing, so I joined the Wing team in 2022.

From my perspective, Winglang is a general purpose language that aims to bridge the gap between cloud infrastructure configuration and runtime logic.
With its two-phased execution model, its design encourages developers to think holistically about cloud applications, weaving together the definitions of cloud resources and the code that operates within them.
It's a language that has its roots in the pains of writing applications to run on cloud providers like AWS, and draws inspiration from both academic research and real-world industry practices.

In this blog post, I wanted to try and identify what I think are most salient features of Winglang from a language design standpoint.

A comprehensive overview of the language would be too long to fit here, but
if you'd like more of a background on how Winglang came to be, I recommend checking out [this blog post](https://www.winglang.io/blog/2022/11/23/manifesto) by Wing's progenitor [Elad Ben-Israel](https://twitter.com/emeshbi).

Enough blabber - let's get into it.

## Two-phased execution model

Winglang’s programming model is built around the idea that cloud applications are defined by both the configuration of cloud resources and the runtime code which executes inside these resources. These two facades are modeled through the notion of `preflight` and `inflight` scopes.
The top-level of a program begins as a `preflight` scope implicitly, and the `inflight` keyword is used to define blocks of code that may be combined, packaged, and executed later on various compute platforms. 

Inflight functions are able to naturally interact with resources which are defined outside of it, and invoke runtime operations on them.
Inflight function code is also asynchronous by default - operations will be automatically awaited unless explicitly opted out through the `defer` keyword (not implemented yet).

## Cloud services as first-class citizens

Winglang models the notion of cloud resources and services through an object-oriented class hierarchy, where all class objects are named and inserted an in-memory tree. 
This makes it possible to derive a application-unique address or "path" for each cloud resource that is deterministic across compilations - simplifying the process of generating reusable cloud infrastructure configuration.
Polymorphic classes can be written which are concretized into different implementations depending on the developer's target cloud platform.

Winglang classes are uniquely designed for the two-phased execution model: every class can have both `preflight` and `inflight` methods and properties.
Preflight methods are used to define the configuration of the resource, and inflight methods are used to define the runtime logic that executes inside the resource.
The methods you are allowed to access are determined by whether you're in a `preflight` lexical scope or an `inflight` lexical scope.

## Metaprogramming capabilities

At the most basic level, the Wing compiler can synthesize any file or directory structure.
When used for creating cloud applications, these files are typically a set of infrastructure definitions (such as CloudFormation, Terraform or Kubernetes manifests), Dockerfiles, function code bundles, deployment workflows, and any other artifacts that are needed in order to deliver this application to the cloud.

## Immutability by default

Unexpected mutations are a common source of programmer error in large-scale systems, especially when sharing data across abstraction boundaries. A language-level guarantee that state cannot change offers opportunities for caching and runtime optimizations. To that end, each of Wing’s collection types (`Array`, `Set`, `Map`) has a mutable variant (`MutArray`, `MutSet`, `MutMap`). In addition, variables cannot be reassigned to unless it is opted into with the `var` keyword.

## No null pointers

The initialization state of every variable in Wing can be determined by its type, and every variable must be initialized before it can be used. A common use of null in other languages, to represent an empty state or sentinel value, is subsumed by the language’s more general-purpose Optional type (typically notated as `T?`), and a selection of syntaxes for easily manipulating Optional values.

## Interop with TypeScript/JavaScript

To avoid building an entire language ecosystem from scratch, Wing assumes a JavaScript-based runtime environment, both for dynamically generating configuration files and as the default way to run inflight application code. This makes it straightforward to reuse code from npm's ecosystem of 3M+ packages, and to “escape hatch” into the underlying JavaScript runtime environment for cases where Wing’s abstractions are not sufficient.

## JSON as a first-class type

JSON is one of the most widely used protocols for sending and receiving information between internet services due to its simplicity and human-readability. As the lingua franca of the cloud, Wing includes it as a built-in data type, with `Json` and `MutJson` variants. Some of the features enabled by the JSON type to make life easier for developers are schema validation and the ability to parse JSON values into well-typed structs with compiler support.

## Local type inference

To save some quantity of programmer key-pressing, Wing supports local type inference: signatures of functions and classes always require type annotation, but within the body of a function, many variables can be declared without any annotations, and Wing will infer the variable’s type from its initialization or its usage.

## Object-oriented development

Classes and interfaces are the primary abstraction mechanisms for both composing data and behavior together, and for sharing APIs between different projects or Wing libraries. While OO has gotten a bad reputation in various circles due to some languages compelling the use of verbose design patterns, Wing learns from its predecessors to provide an expressive development experience that does not impose excessive boilerplate or incidental complexity.

## Library-author friendly

All structures defined by the user, such as classes, interfaces, enums, structs, methods, and properties, are private for sharing across module boundaries unless explicitly declared public. The language toolchain will have utilities for comparing the API surface of versions of libraries, and automatically detecting type-level breaking changes to public APIs for enforcing semver compatibility.

## Algebraic data types

Wing contains the usual assortment of algebraic types from functional languages, like function types, tuples, disjoint union types, and structs. It will be possible to easily destructure algebraic types into their components, and pattern match them using the `switch` keyword.

## Generic code

Wing will support a number of forms of parametric polymorphism, such as classes, iterators, and functions parameterized by other types.
