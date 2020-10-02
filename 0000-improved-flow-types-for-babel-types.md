- Repo: `babel/types`
- Start Date: 2020-10-02
- RFC PR: <!-- leave this empty, to be filled in later -->
- Related Issues: <!-- if relevant -->
- Authors: Micha Reiser
- Champion:
- Implementors: Micha Reiser

# Summary

Create a more precise type definition of `babel/types` for Flow with better type refinement support by declaring the node types as `type`s instead of classes and `BabelNode` as a union type.

# Basic example

```
declare type BabelNodeIdentifier = {|
  leadingComments?: Array<BabelNodeComment>;
  innerComments?: Array<BabelNodeComment>;
  trailingComments?: Array<BabelNodeComment>;
  start: ?number;
  end: ?number;
  loc: ?BabelNodeSourceLocation,
  type: "Identifier";
  name: string;
  decorators?: Array<BabelNodeDecorator>;
  optional?: boolean;
  typeAnnotation?: BabelNodeTypeAnnotation | BabelNodeTSTypeAnnotation | BabelNodeNoop;
|};

declare type BabelNodeIfStatement = {|
  leadingComments?: Array<BabelNodeComment>;
  innerComments?: Array<BabelNodeComment>;
  trailingComments?: Array<BabelNodeComment>;
  start: ?number;
  end: ?number;
  loc: ?BabelNodeSourceLocation,
  type: "IfStatement";
  test: BabelNodeExpression;
  consequent: BabelNodeStatement;
  alternate?: BabelNodeStatement;
|};

....

declare type BabelNode = BabelNodeIdentifier | BabelNodeIfStatement | ...
```

Nodes can then be refined by the `node.type`

```
import * as t from '@babel/types';
import type {StringLiteral, NumericLiteral} from '@babel/types';

function asStringLiteral(ast: ?StringLiteral | NumericLiteral): BabelNodeStringLiteral | null {
  if (ast == null) {
      return t.stringLiteral("null");
  }

  if (ast.type === "StringLiteral") {
    return ast;
  }

  return t.stringLiteral(String(ast.value));
}
```

[Complete Type Definition](https://gist.github.com/MichaReiser/8d8784b4b9e9d26572f817e44aa6bc62)

# Motivation

The current flow typing defines each node type as a `class` and specifies type refinements (`%checks (node instanceof MemberExpression)`) for the `is*` methods.

```
const {isIdentifier = require('@babel/types');

if (isIdentifier(node)) {
  console.log(node.name);
}

```

The todays system has two shortcomings:

* Exporting the node types as `class`es is semantically incorrect. `@babel/types` does not implement the node types as `class`es nor does it export any value for each type. Declaring the node types as `class`es might misguide users to check for a certain node type using `node instanceof Identifier` or to create a new node by calling the constructor `new Identifier()`.
* Flow only supports `%checks` refinements on function declarations and that only if the function isn't called as a method. For example, `isIdentifier(node)` is supported but `t.isIdentifier(node)` (method invocation) or `path.isIdentifier()` are not. Having to import each `is*` function is cumbersome and declaring the `is*` methods on `path` isn't possible with what Flow supports today.


# Detailed design

The proposal of this RFC is to change today's typing to:

* Use exact types instead of `class`es to make it clear that `@babel/types` does not export values for node types.
* Use a union type for `BabelNode` rather than a base class. This allows to refine on node types using `node.type === 'Identifier'` or paths `path.node.type === 'Identifier'`.
* Change the `%checks` on the `is*` function declarations from `node instanceof XNode` to `node != null && node.type === 'X'`

Example Usage:

```
import * as t from '@babel/types';
import type {StringLiteral, NumericLiteral} from '@babel/types';
import {isStringLiteral} from '@babel/types`;

function asStringLiteral(ast: ?StringLiteral | NumericLiteral): BabelNodeStringLiteral | null {
  if (ast == null) {
      return t.stringLiteral("null");
  }

  if (isStringLiteral(ast)) { // or ast.type === "StringLiteral"
    return ast;
  }

  return t.stringLiteral(String(ast.value));
}
```

This changes can be implemented by changing the type definition generation script in `babel/types` for flow only.

## Regression
* The node types can no longer be referenced in `%check` declarations. E.g. `%checks (node instanceof BabelNodeIdentifier)` is no longer valid because `BabelNodeIdentifier` is now a type and not a class. The refinement must be rewritten to `%checks (node != null && node.type ==='Identifier')`
* Referencing node types in a value position is a compile error. For example `import {BabelNodeIdentifier} from '@babel/types';` is no longer valid because `@babel/types` does not export such value. It must be changed to `import type {BabelNodeIdentifier} from '@babel/types'`.


# Drawbacks

* Braking Change:
    * References to the node types as values must be rewritten to type references
    * `%checks` declaration in consumer code must be rewritten to refine on `type`.
* Unions can potentially be expensive during type checking. Flow does some optimisations and I tried out the my definition file and haven't noticed a regression compared to the existing type definition.
* Union type error messages can be hard to understand because flow lists each possible option.

# Alternatives

## Keep as is

The drawbacks are:

* Having to import the `is*` methods for type refinement to work
* Defining the node types as classes is misguiding for users that the node types are modelled as values (classes) when they're not
* Type refinements on `path` using `is*` or `path.type === 'Identifier'` are not supported.

## Inexact vs Exact Types

An alternative is to define the node types either as inexact types or interfaces. The difference is this would allow consumers to set additional props on the nodes.

I believe that exact types are the right choice here since additional properties are e.g. not supported by `babel/generate` and `@babel/traverse` offers `NodePath.setData/getData` to store custom data.

# Adoption strategy

## Create a transform script

Create a babel plugin that rewrites the `%checks` refinements and changes the imports of node types from values to types.

## Export old typing to flow-typed

1. Publish the old flow types on [flow-typed](https://github.com/flow-typed/flow-typed) so that users can ping the version
2. Update the type definition in `babel/types`.

There's still disturbance since consumers will have to upgrade at some point if they want to use newly added node types.

# How we teach this

I don't believe there's any teaching needed nor any need to re-organize the documentation since most remains as is.

# Open questions

* Should the global `BabelNode*` exports be replaced by module level exports without the `BabelNode` prefix to better align with the type script types or should the definition file export both until the next major version.

## Frequently Asked Questions


## Related Discussions

* [PR](https://github.com/babel/babel/pull/12135)
