# Feature testing in import maps
## Or: CSS `@supports` for JavaScript and WebAssembly!

Sometimes, a feature is natively implemented in one browser and missing from another. The gap can be made up by polyfills and transpilers, which unfortunately both result in more code being shipped over the network and parsed by the browser, and generally slower startup time. This proposal provides a tool to serve different JavaScript to different browsers based in feature tests which live outside of the JavaScript code. These feature tests are *declarative*, rather than described in JavaScript, so that they can be executed by the browser itself.

The [import maps proposal](https://github.com/wicg/import-maps/) introduces the concept of configurable module specifier resolution based on fallback lists to JavaScript. This proposal uses that fallback list to decide how to implement a module based on testing for the existence of features.

## Example scenarios

*All browser names have been changed in the following examples; any resemblance to currently developed browsers is purely accidental.*

Let's imagine that there are three popular rendering engines under active development: Mosaic, Lynx, and HotJava. They're all doing a great job participating in web standards, implementing emerging standards when it makes sense, rapidly distributing new browser versions to web users, etc. However, sometimes, one ships a feature before another.

The X Framework aims to work well on Mosaic, Lynx and HotJava. It's really excited about emerging standards, so it aims to make use of them as soon as possible, falling back to polyfills and transpiled code when necessary.

### `import()`

Mosaic and Lynx have been shipping the [dynamic import](https://github.com/tc39/proposal-dynamic-import) proposal for a year, but HotJava has not yet implemented it yet. The X framework wants to use `import()` for code splitting when possible, since the polyfill results in slower loading. However, as `import()` is a syntax error in HotJava, it can't simply use it behind a JavaScript conditional. It's also context-dependent and hard to move from one file to another, given how relative URLs are resolved based on where the module is.

As part of the X Framework's build process, it puts it in two directories: `js-new/` includes native use of `import()`, and `js-old` transpiles away use of `import()`. Accompanying this is the following import map:

```json
{
  "imports": {
    "js/": [
      { "if": { "javascript-syntax": "import(null)" }, "then": "js-new/" },
      "js-old/"
    ]
  }
}
```

### `Intl.RelativeTimeFormat`

HotJava was the first to implement and ship the entire [`Intl.RelativeTimeFormat` proposal](https://github.com/tc39/proposal-intl-relative-time/). Then, Lynx shipped most of it, but omitting the [`Intl.RelativeTimeFormat.prototype.formatToParts` method](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RelativeTimeFormat/formatToParts). Meanwhile, it looks like it'll take a little longer to get an implementation done in Mosaic. Framework X wants to use `Intl.RelativeTimeFormat`, but it'd like to avoid shipping a full polyfill to all users. And for Lynx users, it'd be best to ship just the polyfill for the one missing method.

The X Framework's Widget component makes heavy use of `Intl.RelativeTimeFormat`. To support Mosaic, Lynx and HotJava well, it includes both a full polyfill in `"/intl-relative-time-format.mjs"`, as well as a polyfill which just adds `formatToParts` based on the built-in library in `"/intl-relative-time-format-to-parts.mjs"`. There's an empty module at `"/empty.mjs"`. These are selected as follows:

```json
{
  "imports": {
    "intl-relative-time-format": [
      { "if": { "global": "Intl", "property": "RelativeTimeFormat.prototype.formatToParts" }, "then": "/empty.mjs" },
      { "if": { "global": "Intl", "property": "RelativeTimeFormat" }, "then": "/intl-relative-time-format-to-parts.mjs" },
      "/intl-relative-time-format.mjs"
    ]
  }
}
```

Before using `Intl.RelativeTimeFormat`, the polyfill is loaded with the statement, `import "intl-relative-time-format"`.

### `BigInt` and `BigInt.prototype.toLocaleString()`

Lynx implemented and shipped [`BigInt`](https://github.com/tc39/proposal-bigint/) proposal, but omitted the `BigInt.prototype.toLocaleString` method because the specification was not yet mature. Later, the method shipped in Lynx, but old versions of Lynx remain in broad use.

A developer wants to create a calculator widget, which can operate accurately on large integers, and needs to output them in a locale-dependent way. The X Framework encourages developers to write code using [JSBI](https://github.com/GoogleChromeLabs/jsbi), then transpiles it to native `BigInt` for browsers that support it using [babel-plugin-transform-jsbi-to-bigint](https://github.com/GoogleChromeLabs/babel-plugin-transform-jsbi-to-bigint).

The widget is included using `<script type=module src="/calculator.mjs"></script>`. `/calculator.mjs` contains the code using BigInt, and starts with the line `import  "/bigint-to-locale-string-polyfill.mjs"`. The code based on JSBI is in "/calculator-jsbi.mjs".

```json
{
  "imports": {
    "/calculator.mjs": [
      { "if": { "javascript-syntax": "0n" }, "then": "./calculator.mjs" },
      "/calculator-jsbi.mjs"
    ],
    "/bigint-to-locale-string-polyfill.mjs": [
      { "if": { "global": "BigInt", "property": "prototype.toLocaleString" }, "then": "/empty.mjs" },
      "/bigint-to-locale-string-polyfill.mjs"
    ]
  }
}
```

### Temporal `Duration` type

The initial [`std:temporal`](https://github.com/tc39/proposal-temporal) module ships, initially, exporting five classes: `Instant`, `ZonedInstant`, `DateTime`, `Date` and `Time`. Mosaic, Lynx and HotJava implementations are released around the same time, based on a single implementation in JavaScript which can be easily retargeted to multiple browsers. One year later, the need for a `Duration` type is noted, and it is added as a sixth named export of the `std:temporal` module.

Unfortunately, even though `std:temporal.Duration` is released rapidly to users of new browsers, a significant portion of the web still uses various older versions of Lynx and HotJava, taking time to update. There are many users out there who are missing `std:temporal` entirely, while others have the five exports but not the sixth.

The X Framework makes use of `std:temporal` all over the place, and quickly adopted the new `Duration` feature.

```json
{
  "imports" {
    "std:temporal": [
      { "if": { "module": "std:temporal", "exports": "Duration" }, "then": "std:temporal" },
      { "if": { "module": "std:temporal" }, "then": "./duration-wrapper.mjs" },
      "./full-temporal-polyfill.mjs"
    ]
  },
  "scopes": {
    "/duration-wrapper.mjs": {
      "std:temporal": "std:temporal"
    }
  }
}
```

`/duration-wrapper.mjs` contains `export * from "std:temporal"; export class Duration { /* ... */ }`.

### WebAssembly bulk memory instructions

The [Bulk Memory Operations Proposal for WebAssembly](https://github.com/WebAssembly/bulk-memory-operations) adds more efficient operations for zeroing a large range, copying memory, etc. These are things that were already possible to implement previously with smaller instructions, but can be done more efficiently across large ranges with a special intrinsic.

In WebAssembly, features can be tested imperatively using the [`WebAssembly.validate`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/validate) method. Small WebAssembly programs can be validated to test if they include only supported opcodes. These results can be used to inform which module is pulled in with [`WebAssembly.instantiateStreaming`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiateStreaming).

In the context of the [WebAssembly/ESM integration proposal](https://github.com/webassembly/esm-integration), module specifiers can directly map to WebAssembly modules. When importing a WebAssembly module as an ES module, there's no particular chance to do these imperative `validate` checks.

The X Framework has an image processing component which needs to `memcpy` some large ranges. It has compiled two different WebAssembly modules, one which uses the new feature `"/image.wasm"` and one which does not `"/image-legacy.wasm"`. Imagine that the string `"nf0q29843n0vq340nfwe"` is the result of base-64 encoding a WebAssembly module which exercises the `memcpy` feature. The image processing component would include the following in its import map, to choose the right Wasm module:

```json
{
  "imports": {
    "/image.wasm": [
      { "if": { "wasm-valid": "nf0q29843n0vq340nfwe" }, "then": "/image.wasm" },
      "/image-legacy.wasm"
    ]
  }
}
```

### Customized built-in elements

Everyone's talking about custom elements, and Framework X works to build on them as closely as possible, helping it to be lightweight, efficient and composable. Mosaic, Lynx and HotJava all implement [autonomous custom elements](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements#Autonomous_custom_elements), but [customized built-in elements](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements#Customized_built-in_elements) is only supported by Mosaic and HotJava at the moment. The X Framework wants to use customized built-in elements for improved initial render time and accessibility in its component library, and use alternative, lengthy, component-specific JavaScript logic when this is not possible.

The X Framework's component library includes some components that, internally, use customized built-in elements. The component `"/component.mjs"` has a fallback `"/component-legacy.mjs"` for when customized built-in elements are not available. The import map includes the following:

```json
{
  "imports": {
    "/component.mjs": [
      { "if": { "global": "customElements", "property": "define", "option": "extend" }, "then": "./component.mjs" },
      "./component-legacy.mjs"
    ]
  }
}
```

## Feature queries

This proposal defines a new, JSON-based mini-language to describe whether a particular feature is available in JavaScript.

### Module exists

To check whether a module with a particular module specifier exists, for a module with the specifier `"std:name"`, use:

`{ "module": "std:name" }`

### JavaScript parses

To check whether a particular JavaScript expression parses, for a JavaScript source string (interpreted as a module) `"module"`, use:

`{ "javascript-valid": "module" }`

### WebAssembly validates

To check whether a WebAssembly module is valid, encode the WebAssembly module's binary format in base64, and if that is `"no9atr2aon28afs32"`, then use:

`{ "wasm-valid" : "no9atr2aon28afs32" }`

### Module has an export

To check whether a module `"std:name"` has an export of the name `"exp"`, use:

`{ "module": "std:name", "export": "exp" }`

### Global exists

To check whether a property of the global object `Interface` exists, use:

`{ "global": "Interface" }`

Note that this would include things like attributes operations on the global object, not just interfaces.

### Module export or global has a property

To check whether a module `"std:name"`'s export of the name `"exp"` has a property `"prop"` (i.e., whether `import { exp } from "std:name"; "prop" in exp`):

`{ "module": "std:name", "export": "exp", "property": "prop" }`

This can be used for nested properties (separated by `.`), and properties of globals as well.

The semantics of this are a bit complicated to define (since it should cover both JS-style specs and WebIDL), but it's meant to roughly correspond to, "Would this chain of property accesses exist when run in a new environment?".

### Options bag entries

To check whether a particular method reads a particular property from an options bag, with that options bag passed as the last argument of, e.g., an export `"fn"` from module `"std:name"`, with the option named `"opt"`:

`{ "module": "std:name", "export": "fn", "option": "opt" }`

`"option"` can also be used with globals, properties, etc.

Semantics here are also a bit complicated, but roughly correspond to, "Does this method read a property with this name off of the last parameter of the method?".

## Integration into import maps

In a fallback list in an import map, a new `{ "if": conditional, "then": fallback-list }` construct is supported, with `conditional` being one of the forms listed above, and `fallback-list` being either a single module specifier or an array of them (possibly including further conditionals).

There could also be some syntax for supporting this conditional applying to a broader chunk of the import map, if needed. If you have a use case for that, please open an issue.

## FAQ

### Why not provide an imperative API from JavaScript for these feature tests?

There are already various imperative APIs for these tests. They have certain caveats, but addressing these caveats wouldn't be as powerful as defining a declarative solution. Imperative tests lead to the lose-lose choice of, send the polyfill to the client unconditionally, or insert a later load with something like `document.write` or `import()`--the fetches aren't visible to the browser until some JavaScript is already running, and they come in one by one, leading to slower startup time in the cases where polyfills ar needed. The prefetch scanner could speculate that all of these fetches are needed if it sees them in source, but this speculation could lead to excessive network usage in low-bandwidth situations--which are precisely the times when slow load times are the most acute. This proposal gives the prefetch scanner deeper, declarative knowledge of what's needed, so it can make smarter choices.

### What about testing for bugs in old browsers to fix?

The problem of identifying these historical issues is very complex, but polyfills have already been solving it through runtime tests. It could be hard to define a common lanugage to describe these issues besides the imperative tests. One possibility would be to define a "version number" for each property, and increment it to indicate the lack of a particular known bug. However, coordinating browsers to maintain such a version would be difficult. This refinement is left for a future proposal, and for now, people can continue to use the existing strategy.

### How could these be written in a practical way, integrating with package.json?

The import map can be either hand-written or generated by tools. Jan Krems' [package exports proposal](https://github.com/jkrems/proposal-pkg-exports/) adds a field to packages.json to list exported modules, in a way which can be used to generate import maps. This proposal is designed to fit well into that one; see [discussion](https://github.com/jkrems/proposal-pkg-exports/issues/29).

### Could other features be tested for this way?

Maybe! File an issue and let's discuss it.

### Could this apply in non-web environments?

For other environments where import maps make sense, this proposal might be useful for both JavaScript and WebAssembly. There many be further tests which also make sense, in an environment-specific way. Please file an issue if you have any thoughts about how this proposal applies to non-web environments.

### Why are the tests so granular? What about larger presets?

A [complementary proposal](https://github.com/whatwg/html/issues/4432) by Mathias Bynens and Kristofer Baxter takes this approach. Larger presets could be helpful to reduce the size and complexity of tests sent over, so they may be a very practical option for tooling today. Some reasons why this repository focuses on finer grained tests as well as usage of the results:
- It's difficult to define what the presents contain. Previous attempts (e.g., [hasFeature](https://developer.mozilla.org/en-US/docs/Web/API/DOMImplementation/hasFeature)) didn't go well (browsers lie!).
- There's a lot of interest in shipping code where not all new browsers support the feature yet. Reducing to the lowest common denominaotr wouldn't provide this capability.
- In the context of import maps providing a mechanism for loading polyfills, it makes sense for the polyfill loaded to depend on the features provided by that polyfill, without that necessarily affecting other imports.

If both features are standardized, they could be a great combination: a `srcset` for import maps could be used to reduce the size of the import map delivered to newer browsers, if it gets too big over time with all of this feature testing.

### Why tie this proposal to import maps?

If import maps are the mechanism for loading polyfills of built-in modules, and if we consider it important that finer feature tests lead to how polyfills for built-in modules are loaded, then it's important that these tests work well with import maps.

Import maps already provide the core technology that this proposal builds on: A fallback list used to resolve module specifiers. Import maps give us space to work by being based on a JSON format [which skips over errors](https://github.com/WICG/import-maps/issues/80). It would be a lot of extra work to build this infrastructure in another way, and it wouldn't make much sense to have multiple redundant mechanisms for the same thing on the web.

### How does this proposal interact with APIs which are only exposed in secure contexts, withheld from Workers, etc?

Import maps are always evaluated within a particular JavaScript realm, so the existence of properties, globals, modules, etc. with respect to the [Exposed] and [SecureContext] extended attributes should be well-defined. This existence is already visible to import maps, as kv-storage [may be only present in the module map in secure contexts](https://github.com/WICG/kv-storage/pull/53).

### Why investigate this direction now, rather than waiting to see how import maps pans out in practice?

Whether modules will eventually have feature testing for their contents affects how modules should be designed. For example, if something like this proposal is not adopted, then in the Temporal `Duration` class case, we may want to put `Duration` into a sub-module like `std:temporal/duration` so that it can be polyfilled separately, without needing to load the polyfill in browsers. If that's going to be the eventual shape, then maybe `std:temporal` should be broken up into several tiny modules from the beginning, for each class it exports, for consistency.

However, we believe import maps stands well on its own as an initial feature, and for that reason, this repository describes finer feature tests as a potential follow-on.

### Is this efficiently implementable?

This is unclear. The hope would be that the import map remains interpretable by the network process, when scanning for what to fetch. Asking the network process to understand which JavaScript APIs are available, and to parse JavaScript/validate WebAssembly, would be new, and could increase memory usage. If the queries need to be interpreted by the renderer process, startup time may be slower. However, the reduced volume of code fetched, parsed and executed may compensate. More investigation on implementability is needed before concluding that this feature is feasible.

### Why not just test the User Agent string?

It's often the most practical to do feature testing on the client side, as many polyfills do, and as import maps do as well. Some problems with depending on the UA string:
- Not all layers of the frontend software stack have the ability to reconfigure how serving works.
- UAs can choose their own UA string, and this might not correspond to the features they support.
- Because it can be difficult to deploy UA string testing locally, there are some centralized services like [polyfill.io](https://polyfill.io/v3/) which provide it, but some projects hesitate to depend on external services.
- Browsers have a long history of lying about their UA string in order to influence how UA string testing code acts; encouraging this mechanism more broadly could make the problem worse. Developers have been taught to *not* do this.

### Does this open a new fingerprinting mechanism?

No. This information is already available through existing JavaScript APIs shipped to the web.
