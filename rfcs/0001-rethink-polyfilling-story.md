> :warning: This RFC predates the current process. It has been discussed at [babel/babel#10008](https://github.com/babel/babel/issues/10008).

- Repo: [`babel/babel-polyfills`](https://github.com/babel/babel-polyfills)
- Start Date: 2019-05-21
- Authors: [Nicolò Ribaudo](https://twitter.com/NicoloRibaudo)
- Champion: [Nicolò Ribaudo](https://twitter.com/NicoloRibaudo)
- Implementors: [Nicolò Ribaudo](https://twitter.com/NicoloRibaudo)

# Summary

In the last three years and a half, `@babel/preset-env` has shown its full potential in reducing bundle sizes not only by not transpiling supported syntax features, but also by not including unnecessary `core-js` polyfills.
Currently Babel has three different ways to inject `core-js` polyfills in the source code:
- By using `@babel/preset-env`'s `useBuiltIns: "entry"` option, it is possible to inject polyfills for every ECMAScript functionality not natively supported by the target browsers;
- By using `useBuiltIns: "usage"`, Babel will only inject polyfills for unsupported ECMAScript features but _only_ if they are actually used in the input souce code;
- By using `@babel/plugin-transform-runtime`, Babel will inject po<i>n</i>yfills (which are "pure" and don't pollute the global scope) for every used ECMAScript feature supported by `core-js`. This is usually used by library authors.

> :information_source: **INFO:** Before continuing, I highly recommend reading "[Annex B: The current relationship between Babel and `core-js`](#user-content-annex-b)" to get a deeper understanding of the current situation!

Our position in the JavaScript ecosystem allows us to push these optimizations even further. `@babel/plugin-transform-runtime` has big advantages for some users over `useBuiltIns`, but it doesn't consider target environments: it's 2020 and probably very few people need to load an `Array.prototype.forEach` polyfill.
Additionally, why should we limit this ability to automatically inject only the necessary polyfill to `core-js`? There are also DOM polyfills, Intl polyfills, and polyfills for a myriad of other web platform APIs. Additionally, not everyone wants to use `core-js`: there are many other valid ECMAScript polyfills, which have different tradeoffs (e.g. source size vs spec compliancy) and may work better for some users. 

What if the logic to inject them was not related to the actual data about the available or required polyfills, so that they can be used and developed independently?

# Concepts

- **Polyfill provider** is a special kind of Babel plugin that injects is used to specify which JavaScript expressions need to be polyfilled, where to load that polyfill from and how to apply it. Multiple polyfill providers can be used at the same time, so that users can load, for example, both an ECMAScript polyfill provider and a DOM-related one.
  
  *Polyfill providers* can expose three different methods of injecting the polyfills:
    - `entry-global`, which reflects the current `useBuiltIns: "entry"` option of `@babel/preset-env`;
    - `usage-global`, which reflects the current `useBuiltIns: "usage"` option of `@babel/preset-env`;
    - `usage-pure`, which reflects the current polyfilling behavior of `@babel/plugin-transform-runtime`.

  Every interested project should have their own polyfill provider: for example, `babel-plugin-polyfill-corejs3` or `@ungap/babel-plugin-polyfill`.

# New user experience

Suppose a user is testing some ECMAScript proposals, they are localizing their application using `Intl` and they are using `fetch`. To avoid loading too many bytes of polyfills, they are ok with supporting only commonly used browsers, and they don't want to load unused polyfills.

How would their new config look like?

```json
{
  "presets": [
    ["@babel/env", { "targets": [">1%"] }]
  ],
  "plugins": [
    "@babel/proposal-class-properties",
    ["polyfill-corejs3", {
      "targets": [">1%"],
      "method": "usage-global",
      "proposals": true
    }]
  ]
}
```

# New developer experience

In order to provide consistent APIs and functionalities to our users, we will provide utilities to:
1. centralize the responsibility of handling the possible configuration options to a single shared package
2. abstract the AST structure from the polyfill information, so that new usage detection features will be added to all the different providers.
We can provide those APIs in a new `@babel/helper-define-polyfill-provider` package.

These new APIs will look like this:

```javascript
import definePolyfillProvider from "@babel/helper-define-polyfill-provider";

export default definePolyfillProvider((api, options) => {
  return {
    name: "object-entries-polyfill-provider",

    polyfills: {
      "Object/entries": { chrome: "54", firefox: "47" },
    },

    entryGlobal(meta, utils, path) {
      if (name !== "object-entries-polyfill") return false;
      if (api.shouldInjectPolyfill("Object/entries")) {
        utils.injectGlobalImport("object-entries-polyfill/global");
      }
      path.remove();
    },
    
    usageGlobal(meta, utils) {
      if (
        meta.kind === "property" &&
        meta.placement === "static" &&
        meta.object === "Object" &&
        meta.property === "entries" &&
        api.shouldInjectPolyfill("Object/entries")
      ) {
        utils.injectGlobalImport("object-entries-polyfill/global");
      }
    },
    
    usagePure(name, targets, path) {
      if (
        meta.kind === "property" &&
        meta.placement === "static" &&
        meta.object === "Object" &&
        meta.property === "entries" &&
        api.shouldInjectPolyfill("Object/entries")
      ) {
        path.replaceWith(
          utils.injectDefaultImport("object-entries-polyfill/pure")
        );
      }
    }
  }
});
```

The `createPolyfillProvider` function will take a polyfll plugin factory, and wrap it to create a proper Babel plugin. The factory function takes the same arguments as any other plugin: an instance of an API object and the polyfill options.
It's return value is different from the object returned by normal plugins: it will have an optional method for each of the possible polyfill implementations. We won't disallow other keys in that object, so that we will be able to easily introduce new kind of polyfills, like `"inline"`.

Every polyfilling method will take three parameters:
  - A `meta` object describing the built-in to be polyfilled:
    - `Promise` -> `{ kind: "global", name: "Promise" }`
    - `Promise.try` -> `{ kind: "property", object: "Promise", key: "try", placement: "static" }`
    - `[].includes` -> `{ kind: "property", object: "Array", key: "includes", placement: "prototype" }`
    - `foo().includes` -> `{ kind: "property", object: null, key: "includes", placement: null }`
  - An `utils` object, exposing a few methods to inject imports in the current program.
  - The `NodePath` of the expression which triggered the *polyfill provider* call. It could be an `ImportDeclaration` (for `entry-global`), an identifier (or computed expression) representing the method name, or a `BinaryExpression` for `"foo" in Bar` checks.
 
Polyfill providers will be able to specify custom visitors (like normal plugins): for exapmle, `core-js` needs to inject some polyfills for `yield*` expressions.

# How does this affect the current plugins?

Implementing this RFC won't require changing any of the existing plugins.
We can start working on it as an experiment (like it was done for `@babel/preset-env`), and wait to see how the community reacts to it.

If it will then success, we should integrate it in the main Babel project. It should be possible to do this without breaking changes:
- Remove the custom `useBuiltIns` implementation from `@babel/preset-env`, and delegate to this plugin:
  - If `useBuiltIns` is enabled, `@babel/preset-env` will enable `@babel/inject-polyfills`
  - Depending on the value of the `corejs` option, it will inject the correct polyfill provider plugin.
- Deprecate the `regenerator` and `corejs` options from `@babel/plugin-transform-runtime`: both should be implemented in their own polyfill providers.

# Open questions
- ~~Should the polyfill injector be part of `@babel/preset-env`?~~ No.
  - Pro: it's easier to share the `targets`. On the other hand, it would be more complex but feasible even if it was a separate plugin.
  - Con: it wouldn't be possible to inject polyfills without using `@babel/preset-env`. Currently `@babel/plugin-transform-runtime` has this capability.
  - If it will be only available in the preset, `@babel/helper-polyfill-provider` should be a separate package. Otherwise, we can just export the needed hepers from the plugin package.
  - It could be a separate plugin, but `@babel/preset-env` could include it.
- Who should maintain polyfill providers?
  - Probably not us, but we shouldn't completely ignore them. It's more important to know the details of the polyfill rather than Babel's internals.
  - Currently the `core-js` polyfilling logic in `@babel/preset-env` and `@babel/plugin-transform-runtime` has been mostly maintained by Denis (@zloirock).
  - It could be a way of attracting new contributors; both for us and for the polyfills maintainers.
  - Both the `regenerator` and `core-js` providers should probably be in the Babel org (or at least supported by us), since we have been officially supporting them so far.
- Are Babel helpers similar to a polyfill, or we should not reuse the same logic for both?

# Annex A: The ideal evolution after this proposal

This proposal defines polyfill providers _without requiring any change to the current Babel architecture_. This is an important point, because will allow us to experiment more freely.

However, I think that to further improve the user experience we should:
1. Lift the `targets` option to the top-level configuration. By doing so, it can be shared by different plugins, presets or polyfill providers.
2. Add a new `polyfills` option, similar to the existing `presets` and `plugins`.

By doing so, the above configuration would become like this:
```json
{
  "targets": [">1%"],

  "presets": ["@babel/env"],
  "plugins": ["@babel/proposal-class-properties"],
  "polyfills": [
    ["corejs3", {
      "method": "usage-global",
      "proposals": true
    }]
  ]
}
```

# Annex B: The current relationship between Babel and `core-js`

Babel is a compiler, `core-js` is a polyfill.

A compiler is used to make modern _syntax_ work in old browsers; a polyfill is used to make modern _native functions_ work in old browsers. You _usually_ want to use both, so that you can write modern code and run it in old browsers without problems.

However, compilers and polyfills are two independent units. There are a lot of compiler you can choose (Babel, TypeScript, Traceur, swc, ...), and a lot of polyfills you can choose (`core-js`, `es-shims`, `polyfill.io`, ...).

You can choose them independently, but for [historical reasons](https://github.com/babel/babel/issues/293) (and because `core-js` is a really good polyfill!) so far Babel has made it easier to use `core-js`.

**What does the Babel compiler do with `core-js`?**

Babel internally _does not depend on `core-js`_. What it does is providing a simple way of automatically generate imports to `core-js` in _your_ code.

Babel provides a plugin to generate imports to `core-js` in _your_ code. It's then _your_ code that depends on `core-js`.

```js
// input (your code):
Symbol();

// output (your compiled code):
import "core-js/modules/es.symbol";
Symbol();
```

In order to generate those imports, we don't need to depend on `core-js`: we handle your code as if it was a simple string, similarly to how this function does:
```js
function addCoreJSImport(input) {
  return `import "core-js/modules/es.symbol";\n` + input;
}
```
(well, it's not _that_ simple! :stuck_out_tongue:)

Be careful though: even if Babel doesn't depend on `core-js`, your code will do!

**Is there any other way in which Babel _directly_ depends on `core-js`?**

Kind of. While the Babel compiler itself doesn't depend on `core-js`, we provide a few runtime packages that might be used _at runtime_ by your application that depend on it.

- `@babel/polyfill` is a "proxy package": all what it does is importing `regenerator-runtime` (a runtime helper used for generators) and `core-js` 2. This package has been deprecated _at least_ since the release of `core-js` 3 in favor of the direct inclusion of those two other packages.
    One of the reasons we deprecated it is that many users didn't understand that `@babel/polyfill` just imported `core-js` code, effectively not giving to the project the recognition it deserved.
- (NOTE: `@babel/runtime` contains all the Babel runtime helpers.)
- `@babel/runtime-corejs2` is `@babel/runtime` + ["proxy files"](https://unpkg.com/browse/@babel/runtime-corejs2@7.9.2/core-js/map.js) to `core-js`. Imports to this package are injected by `@babel/plugin-transform-runtime`, similarly to how `@babel/preset-env` injects imports to `core-js`.
- `@babel/runtime-corejs3` is the same, but depending on `core-js-pure` 3 (which is mostly `core-js` but without attaching polyfills to the global scope).

With the polyfill providers proposed in this RFC, we will just generate imports to `core-js-pure` when using `@babel/plugin-transform-runtime` rather than using the `@babel/runtime-corejs3` "proxy".

# Related issues
- https://github.com/babel/babel/issues/9289
- https://github.com/babel/babel/issues/9666
- https://github.com/babel/babel/issues/9842
- https://github.com/babel/babel/issues/9160
- https://github.com/babel/babel/issues/8406
- https://github.com/babel/babel/issues/7052
- https://github.com/babel/babel/issues/2113
- https://github.com/babel/babel/issues/3934
- https://github.com/babel/babel/issues/9363
- https://github.com/babel/babel/issues/6629
