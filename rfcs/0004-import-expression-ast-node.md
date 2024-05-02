- Repo: `babel/babel`
- Start Date: 2020-05-01
- RFC PR: https://github.com/babel/rfcs/pull/4
- Related Issues: https://github.com/babel/babel/pull/10962
- Authors: @JLHwung
- Champion: @JLHwung
- Implementors: @JLHwung

# Summary

This RFC proposes to change the AST shape of dynamic import `import(moduleName)` in ES2020 by introducing a new `ImportExpression` node.

# Motivation

Currently dynamic import is parsed as a `CallExpression`, whose `callee` is a pseudo expression node `Import`. For example, here is the parsed AST of `import("foo")`.

```jsonc
// import("foo")
{
  "type": "CallExpression",
  "callee": {
    "type": "Import"
  },
  "arguments": [
    {
      "type": "StringLiteral",
      "value": "foo"
    }
  ]
}
```

When `@babel/parser` [supported](https://github.com/babel/babylon/pull/163) dynamic import, it was called `import-function` and shared similar syntax structure with `super` call that is parsed as `CallExpression[callee=Super]`. However, it encourages an incorrect mental model, as developers may think of `import()` as a function call. Besides, it was later syntactically limited to one argument, different from other call expressions. Last, it is also problematic when we extend the current AST shape to support stage-1 [module attributes](https://github.com/tc39/proposal-module-attributes) (`import(moduleName, attributes)`) since the arguments (`moduleName` and `attributes` respectively) share the same semantics in the AST.

# Detailed design

Introduce a new `ImportExpression` AST node type in `@babel/types`.

```js
interface ImportExpression <: Expression {
  type: "ImportExpression";
  source: Expression;
}
```

The example above will now be parsed as

```jsonc
// import("foo")
{
  "type": "ImportExpression",
  "source": {
    "type": "StringLiteral",
    "value": "foo"
  }
}
```

## Revision on the `CallExpression` interface

`Import` will be removed from the allowed `callee` type of the `CallExpression` node.

```diff
interface CallExpression <: Expression {
- callee: Expression | Super | Import
+ callee: Expression | Super
}
```

## Revision on the `Import` node constructor

The `Import` node consturctor `t.Import()` will be rendered useless since `Import` is only used as a `callee` type. However, to help new AST adoption, we will preserve `t.Import()` but throws a better message

```js
t.Import()
// throws 
// `t.ImportExpression` should be used when constructing dynamic import `import(moduleName)`
```

## Matching `import()`
Currently a plugin can test if a node is an `import()` by visiting `CallExpression` and filtering by the `callee`.

```js
// Babel 7
export default {
  name: "my-plugin",
  visitor: {
    CallExpression({ node }) {
      if (node.callee.type === "Import") {
        const source = node.arguments[0];
      }
    }
  }
}
```

This RFC proposes `import()` to be parsed a dedicated AST type. Now plugin authors can match `import()` by `ImportExpression`.

```js
// Babel 8
export default {
  name: "my-plugin",
  visitor: {
    ImportExpression({ node }) {
      const source = node.source;
    }
  }
}
```

A plugin can also support both Babel 7 and Babel 8 by checking if `ImportExpression` exists in `@babel/types`.
```js
export default function({ types: t }) {
  let visitor = {};  
  if (t.ImportExpression) {
    // Babel 8
    visitor = {
      ImportExpression({ node }) {
        const source = node.source;
      }
    }
  } else {
    // Compatibility for Babel 7
    visitor = {
      CallExpression({ node }) {
        if (node.callee.type === "Import") {
          const source = node.arguments[0];
        }
      }
    }
  }
  return {
    name: "my-plugin",
    visitor,
  }
}
```

## Constructing `import()`

Currently an `import()` node is constructed via `t.CallExpression`

```js
const t = require("@babel/types");
// import("foo");
t.CallExpression(t.Import(), [t.StringLiteral("foo")]);
```

This RFC adds a new `ImportExpression` constructor

```js
const t = require("@babel/types");
// import("foo");
t.ImportExpression(t.StringLiteral("foo"));
```

Note that `@babel/types` does not currently validate the number of arguments when constructing `import()` AST nodes via `t.CallExpression`. Calling `t.ImportExpression` with more than one argument will throw.


## `module-attributes` proposal support

On top of this RFC, we can add an extra `attributes` property in `ImportExpression` to support the [Module Attributes](https://github.com/tc39/proposal-module-attributes) Proposal.

```ts
interface ImportExpression <: Expression {
  type: "ImportExpression";
  source: Expression;
  attributes: Expression | null;
}
```

Here is an example.
```jsonc
// import("foo", { type: "json" })
{
  "type": "ImportExpression",
  "source": {
    "type": "StringLiteral",
    "value": "foo"
  },
  "attributes": {
    "type": "ObjectExpression",
    "properties": [{
      "type": "ObjectProperty",
      "key": {
        "type": "Identifier",
        "name": "type"
      },
      "value": {
        "type": "StringLiteral",
        "value": "json"
      }
    }]
  }
}
```

We also proposed this approach to ESTree: https://github.com/estree/estree/pull/215


# Drawbacks

## Downstream cost

As with any breaking change to the AST shape, any Babel plugin operating on `import()` will be impacted. For plugins that intend to support both Babel 7 and Babel 8, extra code will be needed.

Introducing a new `ImportExpression` will fail the plugin since it will never be matched without any code modification. Ideally it should be captured by the tests when plugin authors are testing Babel 8 support.

## Implementation complexity

The parser will see extra code in order to parse a new node type. But it should be clearer as `import()` is now separating from `CallExpression`.

# Alternatives

## Stay with the current situation

We can stay with the current situation and use `arguments[1]` to capture module attributes. This does not introduce any breaking changes. Since `import()` was accepted in ES2020, in the future it will become more difficult to introduce any changes to existing AST shapes.

## Add a virtual type `ImportExpression`

`@babel/types` supports virtual types as a shortcut when visiting certain AST nodes. e.g., if we add a new virtual type `ImportExpression` to `CallExpression[callee=Import]`, plugin authors can use `ImportExpression` to match the underhood `CallExpression` node. This approach simplifies the AST node matching but it still contains confusing `arguments` property since the matched node is still a `CallExpression`.

# Adoption strategy

We can ship this AST change behind a `createImportExpressions` boolean parser option in 7.x and encourage downstream projects to test against this feature. In Babel 8 we will remove the flag and enable the new behavior by default.

`t.Import()` will be preserved but it will throw a better message. (See "Detailed Design" above)

Both @devongovett (`parcel`) and @ljharb (`babel-plugin-dynamic-import-node`) have expressed positive feedback to this proposal. (If you know anyone who is interested at this change, please comment and I will reach out)


# How we teach this

We can add migration instructions to the Babel 8 release notes, and also ensure plugin authors add `createImportExpressions` tests to their Babel 7-supporting versions.

# Open questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
    
    If you plan to implement this on your own, what help would you need from the team?
-->

- Do we need a new CI test on all the breaking flags?
