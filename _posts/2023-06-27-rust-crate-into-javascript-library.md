---
title: How to turn a Rust crate into a JavaScript library
---

Hi there - my name's Chris.
I'm an avid software engineer that enjoys building things.

In this post, I'll walk through how I created an [npm] package that re-exports the APIs of a Rust crate for use in JavaScript in a little over an hour.
For demonstration I'll use a crate named [annotate-snippets], which provides an API for pretty-printing error diagnostics like the one shown below - though the technique can be applied to any crate.

_Example diagnostic:_

```
error[E0308]: mismatched types
  --> src/multislice.rs:13:22
   |
13 |         slices: vec!["A",
   |                      ^^^ expected struct `annotate_snippets::snippet::Slice`, found reference
   |
   = note: expected type: `snippet::Annotation`
              found type: `&snippet::Annotation`
```

[npm]: https://www.npmjs.com/
[annotate-snippets]: https://crates.io/crates/annotate-snippets

The main idea will be to use [WebAssembly], a binary instruction format that can be run on many platforms, including browsers and most JavaScript runtimes.
Rust's compiler natively supports compiling Rust programs into WebAssembly. By doing so, we can reuse the capabilities of a Rust crate inside of any JavaScript code - whether it's in the browser, or on another JavaScript runtime like Node.js - without having to rewrite all of the Rust code in a new language.

[WebAssembly]: https://webassembly.org/

## Scaffolding the library

First, start by checking you have recent [Rust] and [Node] installations.

[Rust]: https://www.rust-lang.org/tools/install
[Node]: https://nodejs.org/en/download

Next, create a new directory and add the following files:

### `package.json`

This defines our JavaScript package information:

```json
{
  "name": "my-library",
  "author": "Your name <yourname@example.com>",
  "version": "0.1.0",
  "description": "My WebAssembly bindings",
  "main": "lib/index.js",
  "types": "lib/index.d.ts",
  "files": [
    "*.js",
    "*.d.ts",
    "*.wasm"
  ],
  "keywords": [],
  "license": "MIT",
  "scripts": {
    "build:wasm-pack": "wasm-pack build --target nodejs --out-name index --out-dir ./pkg",
    "build:typescript": "tsc -b",
    "build": "npm run build:wasm-pack && npm run build:typescript",
    "package": "npm pack"
  },
  "devDependencies": {
    "typescript": "5.1.3",
    "wasm-pack": "0.12.0"
  }
}
```

Note that we're actually making a TypeScript library. This is just a JavaScript library with extra type information so users can have documentation and type checking in their IDEs.

### `tsconfig.json`

This defines our TypeScript configuration:

```json
{
  "compilerOptions": {
    "target": "esnext",
    "lib": ["esnext", "dom"],
    "module": "commonjs",
    "declaration": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
  }
}
```

For this post I'll be compiling to "CommonJS" (a standard for how JavaScript code can be packaged into modules), although it should be noted that [ES modules] is a more recent standard that works across more JavaScript runtimes.

[ES modules]: https://hacks.mozilla.org/2018/03/es-modules-a-cartoon-deep-dive/

### `Cargo.toml`

This defines our Rust library information:

```toml
[package]
name = "my-library"
version = "0.1.0"
edition = "2021"

[lib]
# required to compile to WebAssembly
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2.87"
annotate-snippets = "0.9.1"
serde = { version = "1.0", features = ["derive"] }
serde-wasm-bindgen = "0.4"
```

### `.gitignore`

So we don't commit unnecessary stuff to git.

```
/target
/Cargo.lock
/node_modules
*.tgz
```


### `src/lib.rs`

The entrypoint of our Rust project, initialized with a stub function that's exported to WebAssembly:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add(x: usize, y: usize) -> usize {
    x + y
}
```

### `lib/index.ts`

The entrypoint of our TypeScript library, that uses the stubbed function.

```ts
import { add } from "../pkg";

console.log(add(3, 5));
```

### Trying it all together

First, you'll want to build the Rust project by running `cargo build`.
Then, install all of the npm dependencies (including TypeScript and [wasm-pack](https://rustwasm.github.io/wasm-pack/)) by running `npm install`.
Finally, you can generate the WebAssembly bindings and compile the TypeScript library by running `npm run build`.

If everything's working correctly, when you run `node lib/index.js`, it will run the JavaScript code and print out the sum of 3 + 5 calculated in WebAssembly (generated from Rust source code), which is 8.

## Exposing the Rust crate through WebAssembly bindings

Every WebAssembly module defines a set of exports that can be accessed by the host environment.
For our purposes, we just want to export a set of functions that our TypeScript library can call to use to pretty-print diagnostics.

In Rust, we can do this by exporting an appropriate function with the `pub` keyword and annotating it with the `#[wasm_bindgen]` macro.
A function is "appropriate" if the parameters and return values are types that Rust compiler knows how to compile into a WebAssembly representation.
A full list of supported types are described [here].

[here]: https://rustwasm.github.io/wasm-bindgen/reference/types.html

The crate I'm using, `annotate-snippets`, has a simple API that expects a set of options for specifying a code snippet and the errors it should be annotated with, and it can produce a Rust `String` as output. Here's what it would look like to print a snippet in plain Rust:

```rust
let snippet = Snippet {
    title: Some(Annotation {
        label: Some("mismatched types"),
        id: Some("E0308"),
        annotation_type: AnnotationType::Error,
    }),
    footer: vec![Annotation {
        label: Some(
            "expected type: `snippet::Annotation`\n   found type: `__&__snippet::Annotation`",
        ),
        id: None,
        annotation_type: AnnotationType::Note,
    }],
    slices: vec![Slice {
        source: "        slices: vec![\"A\",",
        line_start: 13,
        origin: Some("src/multislice.rs"),
        fold: false,
        annotations: vec![SourceAnnotation {
            label: "expected struct `annotate_snippets::snippet::Slice`, found reference",
            range: (21, 24),
            annotation_type: AnnotationType::Error,
        }],
    }],
    opt: FormatOptions {
        color: true,
        ..Default::default()
    },
};

let dl = DisplayList::from(snippet);
println!("{}", dl);
```

To expose the API through WebAssembly, I'll write a function named `annotate_snippet` that expects the options of the `Snippet` as parameters (title, footer, slices, and formattiong options), and which returns a `String`.

To model these options, we can use an opague type called `JsValue`, and later do some parsing to check the values have the expected structure.

```rust
#[wasm_bindgen]
pub fn annotate_snippet(
    title: JsValue,
    footer: JsValue,
    slices: JsValue,
    options: JsValue,
) -> String {
    todo!()
}
```



## Wrapping the WebAssembly bindings in TypeScript

TODO

## Conclusion

That's all, folks!
