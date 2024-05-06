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

All the plugins for stable ECMAScript features should be moved from the `./packages/babel-plugin-*` folders into `./packages/babel-core/src/internal-plugins/*`.

To make it easier to run tests for an individual plugin (currently you can use `yarn jest transform-classes`), we can still keep tests in separate internal workspaces, for example in `/test/core-plugins/[plugin name]`.

## New Stage 4 proposals

When an ECMAScript proposal is promoted from Stage 3 to Stage 4, we should:
1. Enable it by default in `@babel/parser`
2. Copy the transform plugin into `@babel/core`, and rename it from `@babel/plugin-proposal-*` to `internal:transform-*`
3. Enable it by default in `@babel/preset-env`
4. Archive the `@babel/plugin-syntax-*` syntax plugin
5. Archive the `@babel/plugin-proposal-*` transform plugin

`@babel/preset-env` will only enable plugins supported by the currently used `@babel/core` version. If a proposal reaches stage 4 for the x.y.z Babel release, `@babel/preset-env` will only enable it when both the `@babel/core` and `@babel/preset-env` versions are at least x.y.z. `@babel/preset-env` will not fallback to the `@babel/plugin-proposal-*` plugin when using older `@babel/core` versions, for multiple reasons:
- so that it does not need to depend on `-syntax-` plugins for compatibility with older `@babel/parser` versions;
- to avoid downloading every plugin that reaches Stage 4 twice: once as part of `@babel/core` and once as a dependency of `@babel/preset-env`;
- to encourage people to use the latest `@babel/core` version.

## Bugfix plugins

Bugfix plugins would be subject to the same policy as plugins for whole new features: if they are fixing a bug related to a standard (or stage 4) feature, they should be moved to `@babel/core` named following the `internal:bugfix-[engine name]-[bug description]` pattern.

# Drawbacks

This proposal significantly increases the size of the `@babel/core` package.
However, most of the projects depending on `@babel/core` also depend on `@babel/preset-env` and would already download all the plugins from npm anyway.

If we discover that the additional bundle size causes problems for other tools depending on `@babel/core` but not on `@babel/preset-env`, we could extract and publish two new packages from `@babel/core`:
- `@babel/config-loader`, which handles config loading and plugin/preset resolution;
- `@babel/transform`, which exposes the `transform`, `transformFromAst` and `parse` methods that take a resolved config (ideally that doesn't need FS access) and parse/transform the input.

# Alternatives

## To the whole RFC

- Keep the current status.
- Drop SemVer for the `@babel/core` `peerDependency` of plugins, so that we can require an higher `@babel/core` version when needed. Plugins would then be able to require an higher `@bebel/core` version even without waiting for the next major release.

## To specific parts of the RFC

Instead of the `internalPluginAvailable` function, we can expose an `availableInternalPlugins: ReadonlySet<string>` that `@babel/core` consumers can use to check which plugins are available. This is as simple as `internalPluginAvailable`, with the advantage that it has a standard interface and it also allows iterating over the list of plugin names (is this something that we want to allow? what useages does it unlock?).

# Adoption strategy

For most Babel users, this RFC doesn't require any direct action.

If they depend on one of the packages that will be moved to `@babel/core`, they can remove that dependency and use the `internal:` plugin in their configuraiton.

# How we teach this

Since this RFC doesn't have a big impact on how to use Babel, documenting the new `internal:` plugin is enough.

In the next major release, we can publish an empty placeholder with a deprecation error for all the plugins moved to `@babel/core`, similarly to what we did with `@babel/preset-es2015@7.0.0`.

Additionally, whenever a proposal reaches Stage 4 and is included in `@babel/core` we can `npm deprecate` the previous `@babel/plugin-proposal-*` package, since it will stop being maintained and people should be encouraged to migrate to the `internal:transform-*` plugin.

# Internal migration strategy

_NOTE: This paragraph assumes that we want to implement this RFC in Babel 7. If we end up doing it for Babel 8.0.0, we won't have the "duplicated code" problem._

When mirating to this new approach, we must keep in mind that we have an hard constraint: `@babel/preset-env` must continue being compatible with every `@babel/core` 7.x.y version, and it must continue to enable at least all the plugins that it's currently enabling. This means that we cannot simply move all the plugins to `@babel/core` and remove them from `@babel/preset-env`'s dependencies.

There are three possible approaches that we can take:
1. Copy the existing stable plugins into `@babel/core`, but keep the old ones as dependencies of `@babel/preset-env` so that they can be used as a fallback when using an old `@babel/core` version. This leads to an increased combined size of `@babel/core`+`@babel/preset-env`, since all the plugins' code is effectively duplicated. It will be between around 1MB, less than the total `@babel/preset-env` size, because all the polyfill-related plugins and the compatibility data will _not_ be duplicated.
2. Make `@babel/core` depend on all the existing stable plugins, and re-export them as `internal:transform-*` plugins.
3. Make `@babel/preset-env` depend on a new `@babel/core` version, so that it can access the plugins even if it's being run by an older `@babel/core` version. We might need an intermediate package between `@babel/preset-env` and it's `@babel/core` dependency, because `@babel/core` is already a _peer_ dependency of `@babel/preset-env`.

Approach (2) and (3) lead to the best size outcome (i.e. it won't change), because package managers will continue to be able to deduplicate the plugins packages. However, we must continue assuming that plugins might be used with old `@babel/core` versions and thus they cannot rely on the advantages mentioned in the "Motivation" section above.

Regardess of the approach that we take, new Stage 4 plugins can be directly moved to `@babel/core`, since `@babel/preset-env` doesn't depend yet on them. Additionally, when releasing Babel 8 we can move all the _existing_ stage 4 plugins to `@babel/core` and remove them from `@babel/preset-env`'s dependencies.

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
