- Repo: `babel/babel`
- Start Date: 2021-02-12
- RFC PR: <!-- leave this empty, to be filled in later -->
- Related Issues: <!-- if relevant -->
- Authors: [Nicolò Ribaudo](https://github.com/nicolo-ribaudo)
- Champion: [Nicolò Ribaudo](https://github.com/nicolo-ribaudo)
- Implementors: <!-- the names of everyone who will work on the PR. you can leave this empty if you would like someone else to work on it -->

# Summary

This RFC proposes moving all the existing plugins under the `@babel/` namespace that transform standard ECMAScript syntax into the `@babel/core` package.

From a user perspective this change should be almost invisible, but it gives us better stability and great implementation advantages by drastically reducing the ways in which packages at different versions can be mixed together.

# Motivation

One of the biggest problems that our architecture leads to is that users could mix any version of the "core" packages (`@babel/core`, `@babel/traverse`, `@babel/parser`, etc) with any version of the transform plugins (`@babel/plugin-transform-classes`, `@babel/plugin-proposal-object-rest-spread`, etc).

This is particularly problematic when a plugin version is _ahead_ of the core packages versions.

1. When a proposal moves to Stage 4 and is enabled by default in `@babel/parser`, we still need to defensively enable the syntax plugins until the next major release. For example, the `objectRestSpread` plugin was enabled in `@babel/parser@7.1.0` but `@babel/plugin-proposal-object-rest-spread` still needs to inherit from `@babel/plugin-syntax-object-rest-spread` ([source](https://github.com/babel/babel/blob/c22e72eb24b9dd83d43e693067d9c856acf9977d/packages/babel-plugin-proposal-object-rest-spread/src/index.js#L214)) in case someone is using the plugin with `@babel/parser@7.0.0`.
2. When we introduce a new helper in `@babel/helpers`, we cannot use it safely in any of the transform plugins because users could still have an old `@babel/helpers` dependency. We always need to first check if the helper is defined in the used `@babel/core`/`@babel/helper` version, and fallback to a completely different transformation if it's not ([source](https://github.com/babel/babel/blob/58d2f41930859b5d46001e3ae365ffc09e5a8463/packages/babel-plugin-transform-for-of/src/index.js#L180-L184))
3. When we introduce a new utility function in `@babel/traverse`, we cannot safely rely on it in the transform plugins and we must defensively handle old `@babel/traverse` versions, either by working around fixed `@babel/traverse` bugs ([source](https://github.com/babel/babel/blob/c22e72eb24b9dd83d43e693067d9c856acf9977d/packages/babel-plugin-proposal-object-rest-spread/src/index.js#L6-L15)) or by inlining the new utility in the plugin itself or in a new `@babel/helper-*` package.

# Detailed design

## API

Currently plugin can be specified in one of the following ways:

- `@babel/plugin-[feature]` or `@babel/[feature]`
- `@org/babel-plugin-[feature]` or `@org/[feature]`
- `babel-plugin-[feature]` or `[feature]`
- `module:[package name]`

We can introduce a new protocol similar to `module:`:

- `internal:[feature]`

A configuration to transform classes will thus look like this:

```json
{
    "plugins": ["internal:transform-classes"]
}
```

We also need to export a new function from `@babel/core` that can be used to query the list of available internal plugins:

```ts
export function internalPluginAvailable(name: string): boolean;
```

## Monorepo structure

All the plugins for stable ECMAScript features should be moved from the `./packages/babel-plugin-*` folders into `./packages/babel-core/src/transform-plugins/*`.

To make it easier to run tests for an individual plugin (currently you can use `yarn jest transform-classes`), we can still keep tests in separate internal workspaces, for example in `/test/core-plugins/[plugin name]`.

## New Stage 4 proposals

When an ECMAScript proposal is promoted from Stage 3 to Stage 4, we should:
1. Enable it by default in `@babel/parser`
2. Copy the transform plugin into `@babel/core`
3. Enable it by default in `@babel/preset-env`
4. Archive the `@babel/plugin-syntax-*` syntax plugin
5. Archive the `@babel/plugin-proposal-*` transform plugin

`@babel/preset-env` should still depend on the archived proposal plugin: since it's a separate package, it could still be used with an older `@babel/core` version that doesn't support the new `internal:*` plugin.

It can call, for example, `babel.internalPluginAvailable("internal:transform-classes")` to decide whether it should enable the internal plugin or fallback to the old proposal implementation.

# Drawbacks

This proposal significantly increases the size of the `@babel/core` package.
However, most of the projects depending on `@babel/core` also depend on `@babel/preset-env` and would already download all the plugins from npm anyway.

If we discover that the additional bundle size causes problems for other tools depending on `@babel/core` but not on `@babel/preset-env`, we could extract and publish two new packages from `@babel/core`:
- `@babel/config-loader`, which handles config loading and plugin/preset resolution;
- `@babel/transform`, which exposes the `transform`, `transformFromAst` and `parse` methods that take a resolved config (ideally that doesn't need FS access) and parse/transform the input.

# Alternatives

## To the whole RFC

- Keep the current status.
- Drop SemVer for the `@babel/core` `peerDependency` of plugins, so that we can require an higher `@babel/core` version when needed.

## To specific parts of the RFC

Instead of the `internalPluginAvailable` function, we could export an `internalPluginName` function with the following signature:

```js
export function internalPluginName(name: string): string | undefined;
```

This function takes a plugin name (either internal or external) as the input, and returns the internal plugin name when it's available:

```js
internalPluginName("internal:transform-classes"); // "internal:transform-classes"
internalPluginName("@babel/plugin-transform-classes"); // "internal:transform-classes"
internalPluginName("@babel/plugin-unknown"); // undefined
```

`@babel/core` can then conditionally enable plugins like this:

```js
import transformClasses from "@babel/plugin-transform-classes";

// ...

plugins: [
    babel.internalPluginName("@babel/plugin-transform-classes") ?? transformClasses
]
```

by doing so, we could make old `@babel/preset-env` versions aware of new `@babel/core` internal plugins, so that it doesn't accidentally enable an older proposal implementation after that it becomes Stage 4 and is moved to stage 4.

Is it worth the additional complexity? (we would need a list of plugin names mappings in `@babel/core`).

# Adoption strategy

For most Babel users, this RFC doesn't require any direct action.

If they depend on one of the packages that will be moved to `@babel/core`, they can remove that dependency and use the `internal:` plugin in their configuraiton.

# How we teach this

Since this RFC doesn't have a big impact on how to use Babel, documenting the new `internal:` plugin is enough.

In the next major release, we can publish an empty placeholder with a deprecation error for all the plugins moved to `@babel/core`, similarly to what we did with `@babel/preset-es2015@7.0.0`.

# Open questions

1. If Babel recognizes a plugin for a proposal which has then been moved to `@babel/core`, should it warn? Should it silently replace it with the `internal:` plugin?

2. Should `@babel/core` export these plugins? (e.g. `@babel/core/internal-plugins/transform-classes`)

## Frequently Asked Questions

**Why don't we move proposal plugins to `@babel/core`?**

Proposals are inherently unstable, and can have breaking changes. People shouldn't use them in their applications unless they are willing to keep up to date with the proposal evolution, and we shouldn't make it easier to enable those plugins.

**Why don't we move Flow, TypeScript and JSX plugins to `@babel/core`?**

Many users won't need these plugins (and no one will need both the Flow and the TypeScript plugin at the same time), so we can avoid downloading unnecessary code to their `node_modules`.

Additionally, the Flow and TypeScript plugin aren't usually affected by version problems because their transforms are self-contained (they mostly just delete nodes).

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
