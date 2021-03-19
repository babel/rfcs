- Repo: `babel/babel`
- Start Date: 2020-06-09
- RFC PR: https://github.com/babel/rfcs/pull/6
- Authors: @JLHwung
- Champion: @JLHwung
- Implementors: @JLHwung

# Summary

This RFC proposes to change the AST shape of record property `#{ foo: 42 }` in stage-1 proposal [Record and Tuple] by introducing a new `RecordProperty` node.

# Motivation

Currently a record property is parsed as an `ObjectProperty`, for example, here is the parsed AST of `#{ foo: 42 }`.

```jsonc
// #{ foo: 42 }
{
  "type": "RecordExpression",
  "properties": [
    {
      "type": "ObjectProperty",
      "key": { "type": "Identifier", "name": "foo" },
      "value": { "type": "NumericLiteral", "value": 42 }
    }
  ]
}
```

When `@babel/parser` [supported](https://github.com/babel/babel/pull/10865) record expression, it was reasonable to reuse `ObjectProperty` given that they share same syntax. However when a new proposal [Deep Path Properties in Record Literals] comes up, the current AST can not extend because this proposal has introduced new syntax features (i.e. `#{ foo.bar: 42 }`) only available for record property.

# Detailed design

Introduce a new `RecordProperty` AST node type in `@babel/types`. It has an interface same as `ObjectProperty` except `type` is `RecordProperty`.

```js
interface RecordProperty <: Node {
  type: "RecordProperty";
  key: Expression;
  computed: boolean;
  shorthand: boolean;
  value: Expression;
}
```

The example above will now be parsed as

```jsonc
// #{ foo: 42 }
{
  "type": "RecordExpression",
  "properties": [
    {
      "type": "RecordProperty",
      "key": { "type": "Identifier", "name": "foo" },
      "value": { "type": "NumericLiteral", "value": 42 }
    }
  ]
}
```

This approach is also [adopted](https://github.com/estree/estree/blob/master/experimental/record-tuple.md#recordproperty) by ESTree.

## Revision on the `RecordExpression` interface

`ObjectProperty` will be replaced by `RecordProperty` in allowed `properties` type of the `RecordExpression` node.

```diff
interface RecordExpression <: Expression {
- properties: [ ObjectProperty | SpreadElement ];
+ properties: [ RecordProperty | SpreadElement ];
}
```

## Matching a record property
Currently a plugin can test if a node is an record property by visiting `RecordExpression` its parent path and filtering its `properties` by `[type=ObjectProperty]`.

```js
// Babel 7
export default {
  name: "my-plugin",
  visitor: {
    RecordExpression({ node }) {
      for (const property of node.properties) {
        if (property.type === "ObjectProperty") {
          // property is now a record property
        }
      }
    }
  }
}
```

This RFC proposes record properties to be parsed a dedicated AST type. Now plugin authors can match them by `RecordProperty`.

```js
// Babel 8
export default {
  name: "my-plugin",
  visitor: {
    RecordProperty({ node }) {
      // node is now a record property
    }
  }
}
```

or filtering a `RecordExpression`'s `properties`

```js
// Babel 8 alternative
export default {
  name: "my-plugin",
  visitor: {
    RecordExpression({ node }) {
      for (const property of node.properties) {
        if (property.type === "RecordProperty") {
          // property is now a record property
        }
      }
    }
  }
}
```

A plugin can also support both Babel 7 and Babel 8 by checking.
```js
export default {
  name: "my-plugin",
  visitor: {
    RecordExpression({ node }) {
      for (const property of node.properties) {
        if (
          property.type === "RecordProperty" ||
          property.type === "ObjectProperty" // compatibility for Babel 7
          ) {
          // property is now a record property
        }
      }
    }
  }
}
```

## Constructing `#{ foo: 42 }`

Currently a record expression node is constructed via `t.ObjectProperty`

```js
const t = require("@babel/types");
// #{ foo: 42 };
t.RecordExpression([
  t.ObjectProperty(t.Identifier("foo"), t.NumericLiteral(42), false, false)
]);
```

This RFC adds a new `RecordExpression` constructor

```js
const t = require("@babel/types");
// #{ foo: 42 };
t.RecordExpression([
  t.RecordProperty(t.Identifier("foo"), t.NumericLiteral(42), false, false)
]);
```

## Deep Path Properties in Record Literals proposal support

On top of this RFC, we can extend allowable types of `key` property in `RecordProperty` to support [this][Deep Path Properties in Record Literals] Proposal.

```js
extend interface RecordProperty {
  key: Expression | PropertyPath;
}

interface PropertyPath <: Node {
  type: "PropertyPath";
  computed: boolean;
  ancestor: PropertyPath | Identifier;
  property: Expression;
}
```

Here is an example.
```jsonc
// #{ foo.bar[qux]: 42 }
{
  "type": "RecordExpression",
  "properties": [
    {
      "type": "RecordProperty",
      "key": {
        "type": "PropertyPath",
        "computed": true,
        "property": { "type": "Identifier", "name": "qux" },
        "ancestor": {
          "type": "PropertyPath",
          "computed": false,
          "ancestor": { "type": "Identifier", "name": "foo" },
          "property": { "type": "Identifier", "name": "bar" }
        }
      },
      "value": { "type": "NumericLiteral", "value": 42 }
    }
  ]
}
```

There are various ways to represent `foo.bar` in `#{ foo.bar: 42 }` and it should not be addressed in this RFC. This proposal paves the way for such representations without interfering object properties.


# Drawbacks

## Downstream cost

As with any breaking change to the AST shape, any Babel plugin operating on record properties will be impacted. For plugins that intend to support both Babel 7 and Babel 8, extra code will be needed.

Except for the experimental polyfill [record-tuple-polyfill], the practical cost is considerred low given that `RecordExpression` was not supported until 7.9.0.

## Implementation complexity

The parser will see extra code in order to parse a new node type.

# Alternatives

## Stay with the current situation

We could stay with the current situation and use `ObjectProperty` to capture record properties. This does not introduce any breaking changes, but extra validations should be done when parsing deep record properties path because deep properties is not allowed in object literals.

# Adoption strategy

TBD.

# How we teach this

TBD.

# Open questions

TBD.

[record-tuple-polyfill]: https://github.com/bloomberg/record-tuple-polyfill
[Deep Path Properties in Record Literals]: https://github.com/tc39/proposal-deep-path-properties-for-record
[Record and Tuple]: https://github.com/tc39/proposal-record-tuple
[ESTree Record Tuple]: https://github.com/estree/estree/pull/218