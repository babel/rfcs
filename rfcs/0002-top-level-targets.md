- Repo: `babel/babel`
- Start Date: 2020-04-25
- RFC PR: [babel/babel#12189](https://github.com/babel/babel/pull/12189)
- Related Issues: [babel/babel#8809](https://github.com/babel/babel/pull/8809), [babel/babel#10008](https://github.com/babel/babel/issues/10008)
- Authors: [@nicolo-ribaudo](https://github.com/nicolo-ribaudo) 
- Champion: [@nicolo-ribaudo](https://github.com/nicolo-ribaudo) 
- Implementors: [@nicolo-ribaudo](https://github.com/nicolo-ribaudo) 
- Released in Babel 7.13.0

# Summary

This RFC proposes moving the [Browserslist](https://github.com/browserslist/browserslist) integration from `@babel/preset-env` to `@babel/core`. This means deprecating all the Browserslists-related option from the preset and introducing new top-level configuration options.

# Basic example

This example shows how a simple existing configuration can be rewritten using the new API.

```json5
// Existing
{
  "presets": [
    ["@babel/preset-env", {
      "targets": ">1%, not ie 11"
    }]
  ]
}
```

```json5 
// Proposal
{
  "targets": ">1%, not ie 11",
  "presets": ["@babel/preset-env"]
}
```

# Motivation

`targets` is the option that makes `@babel/preset-env` the most powerful preset we have ever had.

However, there are other plugins and presets that would benefit from knowing what engines the user is targeting. The main motivation for me to write this RFC is to be able to share the same targets between `@babel/preset-env` and the [polyfill plugins]().

Currently, you have to specify the targets twice:
```json5
// Existing
{
  "presets": [
    ["@babel/preset-env", {
      "targets": ">1%, not ie 11"
    }]
  ],
  "plugins": [
    ["polyfill-es-shims", {
      "targets": ">1%, not ie 11"
    }]
  ]
}
```

Moving `targets` to the top-level options can also be useful for:
- plugins that minify the source code, so that they can decide, for exampe, to transform function expressions in arrow functions;
- generic transform plugins, that can emit a smaller output if they know that the user is targeting modern engines.

# Detailed design

`@babel/preset-env` has 3 Browserslist-related options:
- [`targets`](https://babeljs.io/docs/en/babel-preset-env#targets)
- [`configPath`](https://babeljs.io/docs/en/babel-preset-env#configpath)
- [`ignoreBrowserslistConfig`](https://babeljs.io/docs/en/babel-preset-env#ignorebrowserslistconfig)

[babel/babel#11434](https://github.com/babel/babel/pull/11434), which will land in Babel 7.10.0, introduces a new `browserslistEnv` option.

## New Babel options

### `targets`

The `targets` option of `@babel/preset-env` can be used in different ways:
1. As a string ar as an array of strings, to specify a Browserslist query
2. As an object mapping engine names to their target versions. This map can also contain `node: "current"`, to target the Node.js version the users is running Babel with; `esmodules: true`, which is expanded to [the browsers with native support for ECMAScript modules](https://github.com/babel/babel/blob/master/packages/babel-compat-data/data/native-modules.json) and replaces the other targets; `browsers: string | Array<string>`, which can specify a Browserslist query; `uglify: true`, which forces `@babel/preset-env` to compile everything.

We should move this option to `@babel/core` without limiting any of its current capabilities, with the following differences: except for `targets.uglify`.

#### `targets.uglify`

This option shouldn't be ported to `@babel/core`. It has been deprecated since `babel-preset-env` 2.0, for two reasons: if you want to force all the possible syntax transforms regardless of your targets you canuse `@babel/preset-env`'s `forceAllTransform` option, or you can dynamically set your `targets` correctly so that the option reflects what you need.

#### `targets.esmodules`

In `@babel/preset-env` this option completely replaces the other targets: `{ targets: "chrome 40, firefox 70", "esmodules": true }` is resolved to `{ chrome: 61, firefox: 60 }`.
This behavior was a good idea when only recent version of browsers had support for native ECMAScript modules, thus `"esmodules": true` always lifted up the supported targets. However, it now happens that the specified targets are partially more recent than the browsers with native ECMAScript modules support: in this case `"esmodules": true` is unexpectedly causing more features to be compiled.

`@babel/core`'s `targets.esmodules` option will instead always list up the resolved targets, by intersecting the existing targets: `{ targets: "chrome 40, firefox 70", "esmodules": true }` is resolved to `{ chrome: 61, firefox: 70 }`.

### `browserslistConfigFile`

Both the `configPath` and `ignoreBrowserslistConfig` options of `@babel/preset-env` change the Browserslist config loading behavior.
- `configPath` is the file or directory where Browserslist's resolution algorithm starts from. It is the equivalent of Browserslist's [`path` option](https://github.com/browserslist/browserslist#js-api). It defaults to `process.cwd()`.
- `ignoreBrowserslistConfig` disables Browserslist's config resolution algorithm.

Those options conflict with each other, and this is solved by ignoring `configPath` when `ignoreBrowserslistConfig`. However, when the `targets.browsers` option is set, `configPath` takes precedence over `ignoreBrowserslistConfig`.

Given that we are moving them to `@babel/core`, we should consolidate them in a single option: `browserslistConfigFile`. This option would behave exactly like the existing `configFile` option, which controls `babel.config.json`'s loading.

- If set to `false`, it disables config loading for Browserslist.
- If set to a string, it specifies the exact config file path. This is equivalent to Browserslist's [`config` option](https://github.com/browserslist/browserslist#js-api).

If someone needs the old behavior of `configPath`, they can still use Browserslist's config loading API:
```js
// babel.config.mjs
import browserslist from "browserslist";

const configPath = "./folder/to/start/searching/from";

export default {
  presets: ["@babel/preset-env"]
  browserslistConfigFile: browserslist.findConfig(configPath),
};
```

Even if we are removing Browserslist's `path` option from our options, we should still set it internally to allow Browserslist to correctly resolve the config file by default. However, `process.cwd()` (the current default value in `@babel/preset-env`) is a **bad** default. We should choose one of these:
- If Browserslist config files are meant to be project-wide configs, we should use Babel's [`root` option](https://babeljs.io/docs/en/options#root) which represents the root directory of the whole project. It's default value is Babel's [`cwd` option](https://babeljs.io/docs/en/options#cwd), whose default value is `process.cwd()`. Using `process.cwd()` directly would completely ignore the `root` and `cwd` options.
- If Browserslist config files are meant to be file-relative configs, we should default to the input filename. If the input filename is not provided, `browserslistConfigFile` should default to `false` to disable config resolution.

### `browserslistEnv`

The new `browserslistEnv` option of `@babel/preset-env` is the [`env` option](https://github.com/browserslist/browserslist#js-api) of Browserslist. I think we should move it as-is to `@babel/core`'s options.

## Allowed options placements and merging

These three new options are be allowed in the programmatic options, in `babel.config.*` files and in `.babelrc.*` files.

If Browserslist options are specified multiple times, the most relevant one should completely overwrite the previous one. This makes sure that both the following examples have the expected behavior:
```jsonc
{
  "targets": "> ie 10",
  "presets": ["@babel/env"],
  
  "env": {
    "modern": { "targets": "supports es6-module" }
  }
}

{
  "targets": "supports es6-module",
  "presets": ["@babel/env"],
  
  "env": {
    "legacy": { "targets": "> ie 10" }
  }
}
```

If we choosed to merge `targets` by computing the _union_ of the specified targets, then the first example would be resolved to `{ "ie": "10" }` also when `BABEL_ENV=modern`. If we instead choosed to merge them by computing the _intersection_, then the second example would be resolved to the equivalent of `supports es6-module` even when `BABEL_ENV=legacy`.

We should use this precedence, where "a > b" means that the option specified in a overwrites the option specified in b:
- Programmatic options > close `.babelrc.*` > distant `.babelrc.*` > `babel.config.js`

This is identical to what we already do when a plugin is enabled multiple times.

`browserslist()` shouldn't be called every time we find a `targets` or `browserslistConfigFile` option, but only once after merging all the Babel configuration sources.

## Targets resolution time

The `targets` are resolved after loading and merging all the config files, but before returing from `loadPartialConfig`. This guarantees that the object returned by `loadPartialConfig` doesn't rely on additional configuration files (i.e. `.browserslistrc`) or on environment variables that could affect the result of calling `browserslist()`.

## Plugins and presets API

The first parameter passed to plugins and presets (usually named `babel`, or `api`) already exposes different APIs from `@babel/core` to the plugins/presets. [source](https://github.com/babel/babel/blob/83d365acb60db2943279cb6f3914c55f52b5702d/packages/babel-core/src/config/full.js#L225-L228).

We can add a `targets` **method** to that object containing the _resolved_ targets. For example, if the user specifies `targets: ">2%"` in their configuration, a plugin would receive a `targets` object similar to this one:

```js
api.targets() == {
  chrome: "80.0.0",
  ios: "13.3.0",
  safari: "13.0.0",
  samsung: "11.1.0",
}
```

`api.targets()` must be a function because, similarly to `api.env()` and `api.caller()`, it affects plugins and presets caching: `@babel/core` needs to know when a plugin/preset relies on `targets` so that when they change it can properly invalidate the cache and re-instantiate such plugin/preset.

### Internal implementation details

The current implementation uses the [`makeAPI` function](https://github.com/babel/babel/blob/83d365acb60db2943279cb6f3914c55f52b5702d/packages/babel-core/src/config/helpers/config-api.js#L28) to build both the [`plugin API`](https://github.com/babel/babel/blob/83d365acb60db2943279cb6f3914c55f52b5702d/packages/babel-core/src/config/full.js#L227) and the [`config API`](https://github.com/babel/babel/blob/83d365acb60db2943279cb6f3914c55f52b5702d/packages/babel-core/src/config/files/configuration.js#L206). They also share [the same Flow type](https://github.com/babel/babel/blob/83d365acb60db2943279cb6f3914c55f52b5702d/packages/babel-core/src/config/helpers/config-api.js#L20-L26), even if it's called `PluginApi`.

Since `targets` is resolved _after_ loading the config files, we can only pass it to the plugins and presets. We should split the `makeAPI` function in `makePluginAPI` and `makeConfigAPI`, and we should add a `ConfigApi` Flow type. Since this is an internal file, it won't be a breaking change.

# Drawbacks

- This proposal increase the downloaded `node_modules` size even for users who use `@babel/core` without using `@babel/preset-env`. It won't increase `@babel/standalone`'s size, because it already bundles Browserslist.

- We already have two top-level options containing `env` in their name. After this RFC, we would have `env`, `envName` and `browserslistEnv`. They are all different:
    - `envName` is used to overwrite the `BABEL_ENV` and `NODE_ENV` environment variables. It can only be used in the programmatic options, because it affects the config loading behavior.
    - `env` is used to specify custom configs depending on the value of the `envName` option, or on the `BABEL_ENV` and `NODE_ENV` environment variables.
    - `browserslistEnv` is similar to `envName`, but it can also be set in config files.

- Some presets (for example, [`babel-preset-react-app`]((https://github.com/facebook/create-react-app/blob/e89f153224cabd67efb0175103244e0b7f702767/packages/babel-preset-react-app/dependencies.js#L74-L76))) include `@babel/preset-env`, and specify some targets.
  If we move the `targets` option from `@babel/preset-env` from `@babel/core`, then they won't be able to easily specify it anymore because presets would be "`targets` consumers" and not "`targets` providers".

  How big is this drawback? `create-react-app` can move the `targets` logic from the preset to the generated Babel config, but are there any other presets that are defining `targets` for the user and passing it to `@babel/preset-env`?

  Note that, as a workaround, it would be possible to overwrite the `targets` for any plugin or preset by using an higher order function:

  ```js
  export default {
    targets: [">2%"],
    presets: [
      withTargets("@babel/preset-env", { chrome: "42.0.0" })
    ]
  };

  function withTargets(pkg, targets) {
    const fn = require(pkg).default;
    return (api, options, dirname) =>
      fn({ __proto__: api, targets: () => targets }, options, dirname);
  }
  ``` 

# Alternatives

## Keep the current situation

Now that we have extracted the `@babel/helper-compilation-targets` package out of `@babel/preset-env` ([babel/babel#10899](https://github.com/babel/babel/pull/10899)), it's easy for plugin and presets authors add the `targets` option to their packages without needing a new top-level option. However, this requires the users to specify their targets multiple times.

## Merge the new options in a single object

Instead of adding `targets`, `browserslistEnv` and `browserslistConfigFile`, we could decide to have a single `browserslist` option that includes them. We could then use `envName` and `configFile` as the option names, to exactly match the names we alread have for Babel:

```json5
{
  "browserslist": {
    "targets": ">2%",
    "envName": "production",
    "configFile": "./.browserslistrc"
  }
}
```

However, this has two drawbacks:
1. If you explicitly set your `targets` using the "name -> version" object map, it won't actually call Browserslist so the name might be misleading.
2. Most users will only use the `targets` option, and having to set `browserslist.targets` instead of `targets` seems like an unnecessary (even if very low) barrier:
    ```json5
    {
      "browserslist": {
        "targets": ">2%"
      }
    }
    
    // vs
    
    {
      "targets": ">2%"
    }
    ```

We could solve 2 by allowing a top-level `targets` option as an alias to `browserslist.targets`, or by only putting `envName` and `configFile` inside the `browserslist` object.

# Adoption strategy

We can implement this in a non-breaking way: we can add the new top-level options without removing the counterparts in `@babel/preset-env`. We can mark the existing options as deprecated in the docs, and remove them either in the next major version or in the following one.

During this migration period, we can make `@babel/preset-env` throw if the options are duplicated inside the preset and in the top-level.

# How we teach this

We have a problem here: almost all the documentation, articles and tutorials about `@babel/preset-env` show (correctly) the `targets` option. We will update our documentation to document the new top-level options instead of the current one, but this will not be enough.

In the next major release, if we won't remove the `@babel/preset-env` options yet, we should add a warning asking the user to use the top-level options and explaining the benefits. The benefits will be much more clearer when we'll extract the polyfills injection logic from `@babel/preset-env` to a separate plugin.

When we will finally remove the options from `@babel/preset-env`, if they are still set we shouldn't ignore them but throw an error explaining the new top-level options.

# Open questions

1. Should `.browserslistrc` files be considered as project-wide or file-relative configs?
   [**ANSWER**](https://github.com/babel/rfcs/pull/2#issuecomment-619573888): They are file-relative config files.
2. Is the "Merge the new options in a single object" better than having three new top-level options?
   **ANSWER**: No, most configs will use at most one of these options anyway. 
3. Should we also move the `forceAllTransform` option to `@babel/core`?
   **ANSWER**: No, it's an option speficially for syntax transforms, necessary to feed Babel's output into tools that only support ES5 syntax.

<!-- ## Frequently Asked Questions -->

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Possible follow-ups

- (ht [**@developit**](https://github.com/developit)) We might give access to `@babel/compat-data` to plugins through `@babel/core`, possibly with an API similar to this one:
  ```js
  function plugin(api) {
    if (api.targets.supports("proposal-optional-chaining")) {
      /* ... */
    }
  }
  ```
  This should be kept as a possible future RFC because we first need to know:
  - If this would be actually useful
  - What kind of relational operators we need between the config targets and a feature's support data (&ge; and &lt;? However the order isn't always well defined)
  - How does it interact with bugfix-style plugins, that change the resulting targets

## Related Discussions

- [babel/babel#10008](https://github.com/babel/babel/issues/10008), where the "top-level targets" idea first came out.
- Docs of [`babel-polyfills`](https://github.com/babel/babel-polyfills/blob/master/docs/usage.md#targets-ignorebrowserslistconfig-configpath-debug), where I had to duplicate the Browserslist-related options in all the plugins.
- [2020/04/14](https://github.com/babel/notes/blob/master/2020/04-15.md) and [2020/04/02](https://github.com/babel/notes/blob/master/2020/04-02.md) meeting notes, the first times we started thinking about this idea.

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
