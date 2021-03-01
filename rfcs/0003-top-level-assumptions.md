# RFC: Replace "loose" options with a top-level "assumptions" object

- Repo: [`babel/babel`](https://github.com/babel/babel)
- Start Date: 2020-06-08
- RFC PR: [babel/babel#12219](https://github.com/babel/babel/pull/12219)
- Related Issues: <!-- if relevant -->
- Authors: Nicolò Ribaudo
- Champion: Nicolò Ribaudo
- Implementors: Nicolò Ribaudo
- Released in Babel 7.13.0

# Summary

Many of our plugins have a `loose` option, which enables the compiler to make some assumptions about the code you are writing. It uses these to ignore certain edge cases and generate a smaller or faster output.

The `loose` option has been around since `6to5`: at first as a top level option, and then it was split into each plugin as a catch all for anything that we wanted to be less spec compliant.

However, `loose` has some problems: namely the term itself is not really descriptive at all and _should_ be named after the part of the language it affects rather than the plugin that's compiling it (see the "Motivation" section for more details). A lot of this behavior is documented with a single example, so in the future we’ll want to be specific with the assumptions themselves and maybe provide a set of tests/codesandboxes explaining the differences.

This RFC proposes introducing a new *top-level* option, `"assumptions"`, which is an object containing different flags that can simplify the code that Babel generates.
These flags have two characteristics:
- They specify _something_ about a specific language feature, and not about a specific Babel plugin.
- Different flags can toggle different unrelated optimizations in the same plugin, instead of toggling both of them with a single generic `"loose"` option.

# Basic example

```jsonc
// babel.config.json
{
  "assumptions": {
    "noDocumentAll": true,
    "pureGetters": true,
    "iterableIsArray": true
  },
  "presets": ["@babel/preset-env"]
}
```

<details>
<summary>Input / Output</summary>

```javascript 
// input code

for (const el of arr) {
  el.logger?.log("Ok!");
}
```

```javascript 
// output code

for (let i = 0; i < arr.length; i++) {
  const el = arr[i];
  el.logger == null ? void 0 : el.logger.log("Ok!");
}
```

</details>
<br />

```jsonc
// babel.config.json
{
  "assumptions": {
    "noDocumentAll": false,
    "pureGetters": true,
    "iterableIsArray": true
  },
  "presets": ["@babel/preset-env"]
}
```


<details>
<summary>Input / Output</summary>

```javascript 
// input code

for (const el of arr) {
  el.logger?.log("Ok!");
}
```

```javascript 
// output code

for (let i = 0; i < arr.length; i++) {
  const el = arr[i];
  el.logger === null || el.logger === void 0 ? void 0 : el.logger.log("Ok!");
}
```

</details>

# Motivation

The `loose` options have different problems:
- _"What is loose?"_ Since `loose` is not descriptive, it requires users to check a plugin's docs to see what assumptions it makes (and the docs often only partially mention what `loose` means in a specific context).
- _"How loose is loose?"_ Is making `loose` "looser" a breaking change? We have been considering it as such: for this reason, we have been introducing different `loose`-like options in some plugins. For example, `transform-for-of` supports `loose`, `assumeArray` and `allowArrayLike`.
- There are cross-dependencies between the `loose` option of different plugins: for example, `proposal-class-properties`'s loose mode must always match `proposal-private-methods`'s (otherwise we throw an error). This has already caused problems: [babel/babel#11622](https://github.com/babel/babel/issues/11622).
- Some plugins should be aware of other plugins' `loose` option value. For example, we sometimes need to partially transpile optional chaining in the class properties plugin ([babel/babel#11248](https://github.com/babel/babel/pull/11248)). Should we add a new `optionalChainingLoose` option to `proposal-private-methods` and `proposal-class-properties`? Or should we use a cross-plugin communication channel to share it? What about when someone has the private methods plugin enabled, but not the optional chaining one?

This RFC also solves another problem, which is not directly related to the points above:
- `@babel/preset-env` has a `loose` option, which is forwarded to all the plugins it enables. When a user wants to set `loose` only for a specific feature, they have to manually install the plugin and explicitly add it to their configuration ([babel/babel#6978](https://github.com/babel/babel/issues/6978)).


# Detailed design

## New configuration option

This RFC introduces a new top-level option: `assumptions`. This is an object containing different flags to mark different assumptions about the input code as safe and thus allow optimizing them.

```typescript
type Assumptions = {
  [assumption: string]: boolean
};
```

This new option should be allowed in the following locations:
- In programmatic options passed, for example, to `babel.transform()` or to `babel-loader`.
- In `babel.config.json` and `.babelrc.json` top-level options.
- In `overrides` blocks. This is useful, for example, to enable a possible `noDocumentAll` assumption on a folder containing server-side code and not inside one containing client-side code.
- In `env` blocks. This is mostly included for completeness, but I can't think about an use case for it: assumptions are about the _input code_, which isn't affected by env variables.

Additionally, `"assumptions"` should also be allowed inside presets.
There are two main kinds of presets: framework-specific presets, like `babel-preset-react-app`, and company-wide presets, used to share the same plugins accross different projects. Since assumptions describe the input code and not the used plugins, this new option fits better in company-wide presets where the person writing the preset and the person writing the input code have less "degrees of separation", but there are also use cases in framework-specific presets: for example, `babel-preset-react-app` [enables](https://github.com/facebook/create-react-app/blob/c87ab79559e98a5dae2cd0b02477c38ff6113e6a/packages/babel-preset-react-app/create.js#L151-L154) `loose: true` for `@babel/plugin-proposal-class-properties`.

To avoid conflicts between assumptions set in presets, they will only be able to _enable_ them (i.e. set them to `true`). The only way to disable an already enabled assumption is to explicitly do it in a configuration file or in programmatic options. ([discussion](https://github.com/babel/rfcs/pull/5#discussion_r507979771))

## Configuration merging

Multiple `"assumptions"` objects should be merged using `Object.assign`, and not overwritten like other options.
This makes it easy, for example, to enable one additional assumption for a specific folder. Also, it still keeps the ability of disabling an assumption simply by setting it to `false`.

They should be merged considering the following precedence, which is the same as what is already used for other options:
- preset &lt; `babel.config.json` &lt; `.babelrc.json` (far from the compiled file)&lt; `.babelrc.json` (near to the compiled file) &lt; programmatic options
- top-level &lt; `env` section &lt; `overrides` &lt; `overrides.env`

Assumptions set inside a preset are not boxed inside the preset but are applied to all the plugins, for two reasons:
1. conceptually, the assumptions describe the input file and not a specific plugin enabled inside the preset;
2. practically, this makes it possible to have a single "personal" or "company" preset containing assumptions which apply to a user's coding practices, and re-use it in different projects without duplicating the list of assumptions.

In order to expose assumptions defined inside presets to every plugin, we need to first resolve and instantiate all the presets, and then all the plugins with the resolved assumptions. This also means that we cannot provide the defined assumptions list to the presets, because it hasn't been finalized yet when they are instantiated.
This is implemented by [babel/babel#11689](https://github.com/babel/babel/pull/11689)).

## Plugin API

The first parameter passed to the plugins (often known as `api`) should have a new method: `assumptions(name: string): boolean | undefined`, which returns whether or not the assumption has been enabled. If a plugin is asking about an assumption not enabled or not supported by the used `@babel/core` version, it will return `undefined`.

This is implemented as a function and not as an object whose properties reflect the assumptions because it configures the plugins' caching (they are reinstantiated when an assumption they use changes).

## Assumptions list

The different `loose` or `loose`-like options we currently have in our plugins map to these assumptions:

> ℹ️ I'm not particularly attatched to these option names: I tried to choose something descriptive for their behavior, but they can all change.

| Assumption | Behavior | Current option | Repl | Notes |
|:-----------|:---------|:---------------|:----:|:-----:|
| `ignoreToPrimitiveHint` | Transform `` `a${x}b` `` to `"a" + x + "b"` instead of `"a".concat(x, "b")` | `loose` in `transform-template-literals` |
| `mutableTemplateObject` | Don't use `Object.freeze` for the template object created for tagged template literals. This effectively means using the `taggedTemplateLiteralLoose` helper instead of `taggedTemplateLiteral` | `loose` in `transform-template-literals` |
| `ignoreFunctionLength` | The `.length` of a function should be cropped at the first default argument. When this option is enabled, ignore this spec requirement and don't rely on the `arguments` object | `loose` in `transform-parameters` | [🔗](https://babeljs.io/repl#?browsers=defaults&build=&builtIns=false&spec=false&loose=false&code_lz=GYVwdgxgLglg9mABMMAKAhgGkQI0QXkQCZsJsATAxAZgEpEBvAKAF8g&debug=false&forceAllTransforms=false&shippedProposals=true&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=script&lineWrap=true&presets=env%2Cenv&prettier=false&targets=&version=7.10.1&externalPlugins=) |
| `iterableIsArray` | When using an iterable (in array destructuring, for-of or with spreads), assume that the iterable object is an `Array` | `loose` in `transform-destructuring` and `transform-spread`, `assumeArray` in `transform-for-of` | [🔗](https://babeljs.io/repl#?browsers=defaults&build=&builtIns=false&spec=false&loose=false&code_lz=G4QwTgBCELwQ2gOmQIwLoG4BQpLwB4A0EyiAnmrFNkA&debug=false&forceAllTransforms=false&shippedProposals=true&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=script&lineWrap=true&presets=env%2Cenv&prettier=false&targets=&version=7.10.1&externalPlugins=) (it doesn't support `assumeArray`) | 1 |
| `arrayLikeIsIterable` | Allow array-like objects to be used where an iterable is expected. This can be useful, for example, to iterate DOM collections in older browsers | `allowArrayLike` in `transform-destructuring`, `transform-spread` and `transform-for-of` | | 1 |
| `skipForOfIteratorClosing` | When using `for-of` with an iterator, it should always be closed with `.return()` and with `.throw()` in case of an error. This option allows skipping those methods | `loose` in `transform-for-of` | | |
| `objectRestNoSymbols` | When using rest in object destructuring, don't copy symbol keys. This effectively means using the `objectWithoutPropertiesLoose` helper instead of `objectWithoutProperties` | `loose` in `transform-destructuring` and `proposal-object-rest-spread` _(not documented)_ | [🔗](https://babeljs.io/repl#?browsers=defaults&build=&builtIns=false&spec=false&loose=false&code_lz=G4QwTgBA3hIDQQHTIEYQL4QLwQMYG4g&debug=false&forceAllTransforms=false&shippedProposals=true&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=script&lineWrap=true&presets=env%2Cenv&prettier=false&targets=&version=7.10.1&externalPlugins=) |
| `setSpreadProperties` | When using object spread, use `Object.assign` to copy the properties instead of cloning their property descriptors with `Object.defineProperty` | `loose`&`useBuiltIns` in `proposal-object-rest-spread` | |
| `setComputedProperties` | When using computed object properties, use `[[Set]]` semantics (i.e. use an assignment) instead of `[[Define]]` | `loose` in `computed-properties` | [🔗](https://babeljs.io/repl#?browsers=defaults&build=&builtIns=false&spec=false&loose=false&code_lz=G4QwTgBCELwQ3gKAhA2gDwLoC4ICYAaRAXyA&debug=false&forceAllTransforms=false&shippedProposals=true&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=script&lineWrap=true&presets=env%2Cenv&prettier=false&targets=&version=7.10.1&externalPlugins=) |
| `setClassMethods` | When declaring classes, use `[[Set]]` semantics (i.e. use an assignment) instead of `[[Define]]`. This doesn't preserve the correct enumerability. | `loose` in `transform-classes` | [🔗](https://babeljs.io/repl#?browsers=defaults&build=&builtIns=false&spec=false&loose=true&code_lz=MYGwhgzhAECC0G8BQ1oDMD2GAUBKRAvkgUA&debug=false&forceAllTransforms=false&shippedProposals=true&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=script&lineWrap=true&presets=env%2Cenv&prettier=false&targets=&version=7.10.1&externalPlugins=) |
| `setPublicClassFields` | When using computed public class fields, use `[[Set]]` semantics (i.e. use an assignment) instead of `[[Define]]` | `loose` in `proposal-class-properties` | [🔗](https://babeljs.io/repl#?browsers=chrome%2070&build=&builtIns=false&spec=false&loose=false&code_lz=MYGwhgzhAECC0G8BQ1oDMCWBTEATaAvNAIwDcSAvkA&debug=false&forceAllTransforms=false&shippedProposals=true&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=script&lineWrap=true&presets=env%2Cenv&prettier=false&targets=&version=7.10.1&externalPlugins=) |
| `privateFieldsAsProperties` | Instead of storing private fields and methods using a `WeakMap` or a `WeakSet`, define them as own non-enumerable properties of the class instance | `loose` in `proposal-class-properties` _(not documented)_, in `proposal-private-methods` and `proposal-private-property-in-object` | [🔗](https://babeljs.io/repl#?browsers=chrome%2070&build=&builtIns=false&spec=false&loose=false&code_lz=MYGwhgzhAECC0G8BQ1oGIBmBLApiAJtALzQCMA3CtFWgLY4AuAFgPb4AUAlIlQJABOjAK78AdtGZYIAOky4ClVAF8kSoA&debug=false&forceAllTransforms=false&shippedProposals=true&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=script&lineWrap=true&presets=env%2Cenv&prettier=false&targets=&version=7.10.1&externalPlugins=) | 2 |
| `superIsCallableConstructor` | When this option is enabled, `super(arg1)` will be transpiled to `BaseClass.call(this, arg1)`. This means that it won't work with native classes or with built-ins, but only with compiled classes or ES5 constructors | `loose` in `transform-classes` | |
| `constantSuper` | The `super` binding in classes can be changed using `setPrototypeOf`, so it's not possible to statically know it. With this option Babel can assume that the superclass is never changed at runtime. | `loose` in `transform-classes`, `proposal-class-properties`, `proposal-private-methods` and `proposal-decorators`. It should also be added to `transform-object-super`. | [🔗](https://babeljs.io/repl#?browsers=chrome%2040&build=&builtIns=false&spec=false&loose=false&code_lz=MYGwhgzhAEBC0G8BQ1oFsCmAXAFgewBMAKASkWjBAwCcsiAiWAQnpIG5oBfJbpUSGAGFEKdNnzEyCClVoNBLdlx5I-4KNACC0DAA8sGAHYEY8ZKmoBXQ6RGpUAegfQAKgAkAkgGVoAGQ8AcgCiTKKoEJYADjQAdJi4hKRsoty8APIARgBWGMBYMRDYAArUeFhlAJ7RaQBmRJoxkaXlWFUYADTQgo3NldHsqoYYAO5apDFWNuxAA&debug=false&forceAllTransforms=false&shippedProposals=true&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=script&lineWrap=true&presets=env&prettier=false&targets=&version=7.10.1&externalPlugins=) |
| `noClassCalls` | Assume that classes are always instantiated with `new` and that the code never tries to call them as a function. This lets Babel skip the `this instanceof ThisClass` check (`_classCallCheck`). | `loose` in `transform-classes` | |
| `noDocumentAll` | When compiling the `??` and `?.` operators, assume that they are never used with `document.all` and thus `== null` is safe | `loose` in `proposal-optional-chaining` and `proposal-nullish-coalescing-operator` | [🔗](https://babeljs.io/repl#?browsers=chrome%2070&build=&builtIns=false&spec=false&loose=false&code_lz=IYfgdARg3AUMAEIT2kA&debug=false&forceAllTransforms=false&shippedProposals=true&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=script&lineWrap=true&presets=env%2Cenv&prettier=false&targets=&version=7.10.1&externalPlugins=) | 3 |
| `pureGetters` | When an expression that might invoke getters needs to be evaluated multiple times, Babel caches its value. When this option is enabled, the expression can be safely re-evaluated. For example, `a.b?.()` can be compiled to `a.b != null && a.b()` without caching `a.b` | `loose` in `proposal-optional-chaining` and in `proposal-object-rest-spread` _(not documented)_ | [🔗](https://babeljs.io/repl#?browsers=chrome%2070&build=&builtIns=false&spec=false&loose=true&code_lz=IYOgRg_CAUCUQ&debug=false&forceAllTransforms=false&shippedProposals=true&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=script&lineWrap=true&presets=env&prettier=false&targets=&version=7.10.1&externalPlugins=) |
| `enumerableModuleMeta` | When compiling ESM to CJS, Babel defines a non-enumerable, non-witable, non-configurable `__esModule` property on the `exports` object. When this option is enabled, that property is set using a simple assignment | `loose` in `transform-modules-commonjs` |
| `constantReexports` | When re-exporting an imported value, assume that it's value doesn't change and set it with a simple assignment | `loose` in `transform-modules-commonjs`, `transform-modules-umd`, `transform-modules-amd` _(not documented)_ | | |
| `noNewArrows` | Assume that the code never tries to instantiate arrow function using `new`, so Babel can avoid injecting checks to prevent it | `spec` in `transform-arrow-functions` | | 4 |

1. `iterableIsArray` is not compatible with `arrayLikeIsIterable`, even if array-like objects could work with `iterableIsArray` in cases where we only rely on indexed access and not on array methods.
2. Currently, the `loose` option must be the same for these plugins or it will throw an error. We have a workaround in `@babel/preset-env` to allow setting `loose` there differently from these plugins ([babel/babel#11634](https://github.com/babel/babel/pull/11634))
3. `?.` also needs to be compiled by the private fields and methods plugins ([babel/babel#11248](https://github.com/babel/babel/pull/11248), but currently there is no way to set it to `loose`
4. The current default behavior is to consider this assumption as valid, and only produce 100% spec-compliant code when the `spec` option is enabled.

### `loose` features not ported to `assumptions`

- It's not clear if we still need `inheritsLoose` helper injected by the `transform-classes` plugin ([#5 (comment by @jridgewell)](https://github.com/babel/rfcs/pull/5#discussion_r543925453)).

### New assumptions policy

We will only define new assumptions for standard ECMAScript features or for very stable stage 3 proposals (following the same convention we use for `@babel/preset-env`'s `shippedProposals` option).

However, existing assumptions can be used by plugins for proposals in earlier stages of the TC39 process: for example, the `#priv in obj` plugin is likely affected by the same assumptions that affect the class private fields plugin.

# Drawbacks

- This RFC introduces a very big number of new top-level options (20), and makes it relatively cheap to add new ones. Having a big number of options can add more "tooling fatigue" on the shoulders of our users, and it can make it hard for us to properly document them. However, these new options replace 22 existing plugin options, many of which have the same name (`loose`) but different behaviors.

- ([@JLHwung](https://github.com/JLHwung)) Some `assumptions` are only effective in one plugin, but users will have to go through their presets/plugins to see if they have opt-in to this plugins.

# Alternatives

- We could decide to add these new descriptive options names directly in plugins options, instead of in `@babel/core`. This solves the problem of "what does loose mean?", but it doesn't solve the problem of having the same assumption in different plugins.

- We could decide to add a simple top-level `loose` option, like we did in Babel 5: this solves the problem of multiple plugins sharing the same assumption. However, it would be even more obscure than the current `loose` options inside the plugins (which at least give some context). Also, having an all-or-nothing toggle means that often you would have to disable _every_ optimization just because you rely on a single edge case.

# Adoption strategy

We can introduce these options without introducing breaking changes, because it's opt-in. There are three possible compatibility strategies between the existing `loose` options and `assumptions` that we can choose:

1. Allow both, `assumptions` has higher precedence because it's the new recommended option:
    ```javascript
    export default function transformClassProperties(api, options) {
      const { setPublicClassFields = options.loose } = api.assumptions;
    ```

1. Allow both, `loose` has higher precedence because it "nearer" to the plugin:
    ```javascript
    export default function transformClassProperties(api, options) {
      const setPublicClassFields = options.loose ?? api.assumptions.setPublicClassFields;
    ```

1. Disallow using both at the same time. This will make the migration faster but a little harder:
    ```javascript
    export default function transformClassProperties(api, options) {
      let { setPublicClassFields } = options.loose;
      if (options.loose != null) {
        if (setPublicClassFields != null) throw new Error();
        setPublicClassFields = options.loose;
      }
    ```

I prefer the first alternative, since it makes it possible for our users to gradually migrate to the new options and gradually makes the `loose` options noops in users' configs while migrating.

# How we teach this

This RFC proposed the addition of a new set of options in a way that every option can affect multiple plugins, and a plugin can be affected by multiple options.

Documenting the different `loose` and `loose`-like options wasn't too hard: every plugin had a bunch of those options, and we could document them in the plugin's page.

A possible structure for the documentation could be a two-entry table, with all the plugins in the rows and the assumptions in the columns, marking which assumption affects which plugins. However, this would lead to an enormous highly sparse table, impossible to fit in a normal web page.

I think that the best way of documenting these options is to put all of them in a single long list, divided in two sections: assumptions that affect standard features, and assumptions that only affect proposals. For each assumption we should write:
- The option name
- It's description
- An example (which could be interactive, using CodeSandbox)
- The affected plugins

In the docs of each affected plugin, we should write the assumptions that affect it and link to the corresponding entry in the assumptions list. We shouldn't duplicate the description of what each assumption does, because duplicating it would likely lead to the two copies become out of sync.

We can also provide a tool to check that the all the assumptions hold. This can't be done with an ESLint plugin because almost all the assumptions are about runtime semantics hard to statically analyze, but we can do it using a Babel plugin which should run while testing, and which injects assertions in the transpiled code:

```javascript 
// input code

foo?.bar;
```

```javascript
// output code

(() => {
  const _tmp = foo;
  if (_tmp == null && _tmp !== null && _tmp !== undefined) {
    throw new Error(`
      Invalid assumption - noDocumentAll.
      
      "noDocumentAll: true" implies that ?. and ?? are never
      used with the special document.all object.
      This assumption has been violated, so you should
      disable it in your config.
    `);
  }
  return _tmp;
})?.bar
```

<details>
<summary>Config</summary>

```jsonc
// babel.config.json
{
  "assumptions": {
    "noDocumentAll": true
  },
  "env": {
    "test": {
      "plugins": ["@babel/validate-assumptions"]
    }
  }
}
```

</details>

# Open questions

- Should we try to pass to the presets at least a _partial_ `assumptions` object? Currently `@babel/preset-env` relies on `loose` to enable/disable the `typeof-symbol` plugin. **ANSWER:** No, presets can only _produce_ assumptions and not _consume_ them.
- Should we validate the list of `assumptions` in `@babel/core`, and disallow unknown ones? This would make it impossible for third-party plugins to introduce their own assumptions, but it also means that it's easier for us to introduce new assumptions without risking ecosystem incompatibilities. **ANSWER:** Yes, the compatibility problems are easily solved checking Babel's version.
- Should assumptions always default to `false`? Currently everything defaults to being spec-compliant, except for the `arrow-functions` plugin which has a `spec: true` option as an opt-in. **ANSWER:** Yes, we will make `noNewArrows` default to `false` in Babel 8.
- ([@JLHwung](https://github.com/JLHwung)) Should we further infer `assumptions` from `targets`? i.e. `{ targets: "node 8" }` can imply `{ assumptions: { noDocumentAll: true } }`.

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
    
    If you plan to implement this on your own, what help would you need from the team?
-->

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->


## Related Discussions
- [2020-04-02 notes](https://github.com/babel/notes/blob/e4699d49a119877c622255bde259d4dfb84c0d0e/2020/04-02.md)
