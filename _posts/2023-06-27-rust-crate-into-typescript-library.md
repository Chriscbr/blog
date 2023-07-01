---
title: Using WebAssembly to turn Rust crates into fast TypeScript libraries
---

Hi there - my name's Chris.
I'm an avid software engineer that enjoys building things.

In this post, I'll walk through how you can create an [npm] package that re-exports the APIs of a Rust crate for use in TypeScript or JavaScript.
For demonstration I'll use a Apache-2.0 / MIT licensed crate named [annotate-snippets], which provides an API for pretty-printing error diagnostics like the one shown below - though the technique can be applied to any crate.

Here's an example the final library being used, and the formatted diagnostic it prints out:

```ts
import { annotateSnippet } from "annotate-snippets";

console.log(annotateSnippet({
  label: "mismatched types",
  id: "E0308",
  annotationType: "error",
}, [], [{
  source: `) -> Option<String> {
    for ann in annotations {
        match (ann.range.0, ann.range.1) {
            (None, None) => continue,
            (Some(start), Some(end)) if start > end_index => continue,
            (Some(start), Some(end)) if start >= start_index => {
                let label = if let Some(ref label) = ann.label {
                    format!(" {}", label)
                } else {
                    String::from("")
                };

                return Some(format!(
                    "{}{}{}",
                    " ".repeat(start - start_index),
                    "^".repeat(end - start),
                    label
                ));
            }
            _ => continue,
        }
    }`,
  lineStart: 51,
  origin: "src/format.rs",
  fold: false,
  annotations: [
    {
      label: "expected `Option<String>` because of return type",
      annotationType: "warning",
      range: [5, 19],
    },
    {
      label: "expected enum `std::option::Option`",
      annotationType: "error",
      range: [26, 724],
    }
  ]
}], {
  color: true,
  anonymizedLineNumbers: false,
}));
```

(This API isn't beautiful, but it gets the job done.)

And here's the diagnostic it prints out:

```
error[E0308]: mismatched types
  --> src/format.rs:51:6
   |
51 |   ) -> Option<String> {
   |        -------------- expected `Option<String>` because of return type
52 |       for ann in annotations {
   |  _____^
53 | |         match (ann.range.0, ann.range.1) {
54 | |             (None, None) => continue,
55 | |             (Some(start), Some(end)) if start > end_index => continue,
56 | |             (Some(start), Some(end)) if start >= start_index => {
57 | |                 let label = if let Some(ref label) = ann.label {
58 | |                     format!(" {}", label)
59 | |                 } else {
60 | |                     String::from("")
61 | |                 };
62 | |
63 | |                 return Some(format!(
64 | |                     "{}{}{}",
65 | |                     " ".repeat(start - start_index),
66 | |                     "^".repeat(end - start),
67 | |                     label
68 | |                 ));
69 | |             }
70 | |             _ => continue,
71 | |         }
72 | |     }
   | |____^ expected enum `std::option::Option`
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

This defines our JavaScript package metadata:

```json
{
  "name": "my-library",
  "author": "Your name <yourname@example.com>",
  "version": "0.1.0",
  "description": "My WebAssembly bindings",
  "main": "lib/index.js",
  "types": "lib/index.d.ts",
  "files": [
    "lib/**/*.js",
    "lib/**/*.d.ts",
    "pkg/**/*.js",
    "pkg/**/*.d.ts",
    "pkg/**/*.wasm",
    "pkg/**/*.wasm.d.ts"
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

For this post I'll be compiling to "CommonJS" (a standard for packaging JavaScript modules that was designed for NodeJS), although it should be noted that [ES modules] is a more recent standard that works across more JavaScript runtimes.

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
annotate-snippets = { version = "0.9.1", features = ["color"] }
serde = { version = "1.0", features = ["derive"] }
serde-wasm-bindgen = "0.4"
```

If you wanted to create bindings for a different Rust crate, you could easily swap `annotate-snippets` with 

### `.gitignore`

So we don't commit unnecessary stuff to git.

```
/target
/Cargo.lock
/node_modules
*.tgz
```


### `src/lib.rs`

The entrypoint of our Rust project, initialized with a placeholder function that is exported to WebAssembly:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add(x: usize, y: usize) -> usize {
    x + y
}
```

### `lib/index.ts`

The entrypoint of our TypeScript library, that uses the placeholder function.

```ts
import { add } from "../pkg";

console.log(add(3, 5));
```

### Tying it all together

First, you'll want to build the Rust project by running `cargo build`.
Then, install all of the npm dependencies (including TypeScript and [wasm-pack](https://rustwasm.github.io/wasm-pack/)) by running `npm install`.
Finally, you can generate the WebAssembly bindings and compile the TypeScript library by running `npm run build`.

If everything's working correctly, when you run `node lib/index.js`, it will run the compiled JavaScript code and print out the sum of 3 + 5 calculated in WebAssembly (generated from Rust source code), still fortunately 8.

## Exposing the Rust crate through WebAssembly bindings

Every WebAssembly module defines a set of exports that can be accessed by the host environment.
For our example, we just need to export a single function that will generate a pretty-printed diagnostic.

In Rust, we can do this by exporting a function with the `pub` keyword and annotating it with the `#[wasm_bindgen]` macro.
Since we're using `#[wasm_bindgen]`, we need to make sure the function's parameters and return values have types that Rust compiler knows how to compile into a WebAssembly representation - so no fancy higher-kinded types here.
A full list of supported types are described [here].

[here]: https://rustwasm.github.io/wasm-bindgen/reference/types.html

The crate I'm using, `annotate-snippets`, has a simple API.
It expects a set of options for specifying a code snippet and its annotations, and it can produce a Rust `String` as output.
Here's what the using the API in Rust looks like:

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

To make a WebAssembly binding for this, I'll write a function named `annotate_snippet` that expects the options of the `Snippet` as parameters (title, footer, slices, and formatting options), and returns a `String`.

To model the options, we can use an opague type called `JsValue`, and then do some parsing to check the values have the expected structure.

```rust
use wasm_bindgen::prelude::*;

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

Let's start with parsing the first parameter, `options`, which we'll expect to be provided as a plain JavaScript object with fields that match `annotate_snippets::FormatOptions`.
The way we can achieve this is by creating own structs that match the structure of `FormatOptions` that implement the serde `Serialize` and `Deserialize` traits on them.
`serde` is a library that lets you automatically perform conversions between Rust structs and serialized formats.

> Note: We need to create our own structs because Rust [does not allow you to implement foreign traits on foreign types](https://rust-lang.github.io/chalk/book/clauses/coherence.html).

Here are the structs I added:

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct MyFormatOptions {
    color: bool,
    #[serde(rename = "anonymizedLineNumbers")]
    anonymized_line_numbers: bool,
    margin: Option<MyMargin>,
}

#[derive(Serialize, Deserialize, Debug)]
struct MyMargin {
    #[serde(rename = "whitespaceLeft")]
    whitespace_left: usize,
    #[serde(rename = "spanLeft")]
    span_left: usize,
    #[serde(rename = "spanRight")]
    span_right: usize,
    #[serde(rename = "labelRight")]
    label_right: usize,
    #[serde(rename = "columnWidth")]
    column_width: usize,
    #[serde(rename = "maxLineLen")]
    max_line_len: usize,
}
```

For some fields, I've added a macro that renames the field.
The purpose is so that in our TypeScript library, we can use camelCase fields as is the convention in the TypeScript ecosystem, while still representing the fields in Rust using snake_case.

Next, we need to write some glue code for converting these structs back into the corresponding types from the `annotate-snippets` crate.
The idiomatic way to achieve this in Rust by using the `From` trait:

```rust
use annotate_snippets::display_list::{FormatOptions, Margin};

impl From<MyFormatOptions> for FormatOptions {
    fn from(options: MyFormatOptions) -> Self {
        FormatOptions {
            color: options.color,
            anonymized_line_numbers: options.anonymized_line_numbers,
            margin: options.margin.map(|m| m.into()),
        }
    }
}

impl From<MyMargin> for Margin {
    fn from(margin: MyMargin) -> Self {
        Margin::new(
            margin.whitespace_left,
            margin.span_left,
            margin.span_right,
            margin.label_right,
            margin.column_width,
            margin.max_line_len,
        )
    }
}
```

Finally we can use these structs in our `annotate_snippet` function to parse the `JsValue`:

```rust
#[wasm_bindgen]
pub fn annotate_snippet(
    title: JsValue,
    footer: JsValue,
    slices: JsValue,
    options: JsValue,
) -> String {
    let options: FormatOptions = match serde_wasm_bindgen::from_value::<MyFormatOptions>(options) {
        Ok(config) => config.into(),
        Err(_) => {
            return String::from("Error");
        }
    };

    todo!()
}
```

### Error handling

This is great, but if there is a parsing error, it would be nice to give more specific information to the user.

Let's define an external function that will let our Rust code call back into JavaScript and throw an error...

```rust
#[wasm_bindgen(inline_js = "exports.error = function(s) { throw new Error(s) }")]
extern "C" {
    fn error(s: String);
}
```

...and update the code from `annotate_snippet` so we use the `Err` value:

```rust
#[wasm_bindgen]
pub fn annotate_snippet(
    title: JsValue,
    footer: JsValue,
    slices: JsValue,
    options: JsValue,
) -> String {
    let options: FormatOptions = match serde_wasm_bindgen::from_value::<MyFormatOptions>(options) {
        Ok(config) => config.into(),
        Err(err) => {
            error(err.to_string());
            return String::from("Error");
        }
    };

    todo!()
}
```

Nicely done!

Next, we just have to repeat this process for the `title`, `footer`, and `slices` parameters.
The details are a little bit tedious, but the rest of the code is available on GitHub [here](https://github.com/Chriscbr/annotate-snippets/blob/main/src/lib.rs).

When you're finished, you can run `npm run build` to check that it compiles to WebAssembly successfully.

## Wrapping the WebAssembly bindings in TypeScript

At this point, the JavaScript bindings are fully functional.
Here's an example of calling them in `lib/index.ts`:

```ts
import { annotate_snippet } from "../pkg";

console.log(annotate_snippet({
  label: "mismatched types",
  id: "E0308",
  annotationType: "error",
}, [], [{
  source: `) -> Option<String> {
    for ann in annotations {
        match (ann.range.0, ann.range.1) {
            (None, None) => continue,
            (Some(start), Some(end)) if start > end_index => continue,
            (Some(start), Some(end)) if start >= start_index => {
                let label = if let Some(ref label) = ann.label {
                    format!(" {}", label)
                } else {
                    String::from("")
                };

                return Some(format!(
                    "{}{}{}",
                    " ".repeat(start - start_index),
                    "^".repeat(end - start),
                    label
                ));
            }
            _ => continue,
        }
    }`,
  lineStart: 51,
  origin: "src/format.rs",
  fold: false,
  annotations: [
    {
      label: "expected `Option<String>` because of return type",
      annotationType: "warning",
      range: [5, 19],
    },
    {
      label: "expected enum `std::option::Option`",
      annotationType: "error",
      range: [26, 724],
    }
  ]
}], {
  color: true,
  anonymizedLineNumbers: false,
}));
```

The only problem is that by default, all of these fields will be typed as `any` in TypeScript. This means anyone calling the API won't know what fields can be passed in, unless they go to the Rust documentation for the original crate or look at our source code.

We can do better by providing a typed API that shows all of the available fields for each parameter of the `annotate_snippet` function.

Here's our updated TypeScript module in `lib/index.ts`.
All of the types are TypeScript equivalents of the types
from our Rust code (for example, `bool` becomes `boolean`, `Option<String>` becomes optional `string`'s, and simple enums become [string literal types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#literal-types)).

```ts
export type AnnotationType = "error" | "warning" | "info" | "note" | "help";

export interface Annotation {
  id?: string;
  label?: string;
  annotationType: AnnotationType;
}

export interface Slice {
  source: string;
  lineStart: number;
  origin?: string;
  annotations: SourceAnnotation[];
  fold: boolean;
}

export interface SourceAnnotation {
  range: [number, number];
  label: string;
  annotationType: AnnotationType;
}

export interface FormatOptions {
  color: boolean;
  anonymizedLineNumbers: boolean;
  margin?: Margin;
}

export interface Margin {
  whitespaceLeft: number;
  spanLeft: number;
  spanRight: number;
  labelRight: number;
  columnWidth: number;
  maxLineLen: number;
}

export function annotateSnippet(
  title: Annotation | undefined,
  footer: Annotation[],
  slices: Slice[],
  options: FormatOptions,
) {
  return bindings.annotate_snippet(title, footer, slices, options);
}
```

Run `npm run build` again to check everything works.

## Conclusion

That's it! We've successfully wrapped a Rust crate in WebAssembly and TypeScript.

You can see the full source code of the library on GitHub here (https://github.com/Chriscbr/annotate-snippets).

If you have any questions or comments, feel free to reach out to me on [Twitter](https://twitter.com/rybickic) or [GitHub](https://github.com/Chriscbr).
