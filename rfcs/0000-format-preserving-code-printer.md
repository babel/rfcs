# Format-preserving code printer

- Repo: babel/babel, or starting as its own separate repo
- Start Date: 2024-05-02
- RFC PR: <!-- leave this empty, to be filled in later -->
- Related Issues: <!-- if relevant -->
- Authors: Nicolò Ribaudo
- Champion: Nicolò Ribaudo
- Implementors: <!-- the names of everyone who will work on the PR. you can leave this empty if you would like someone else to work on it -->

# Summary

This RFC proposes introducing a new mode in `@babel/generator`, to support printing code by presevring the original format. `@babel/generator` already has a `retainLines` option to preserve line numbers, and this RFC is taking it one step further.

# Basic example

Consider a Babel configuration that transforms the following input code down to ES5:

```js
let x = () => {
  return (
    1 + 2
  ) * 3;
};

console.log(x);

for (var i = 0; i<3; i+=1)
  console.log(i);
```

Currently, Babel generates the following code:
```js
var x = function () {
  return (1 + 2) * 3;
};
console.log(x);
for (var i = 0; i < 3; i += 1) console.log(i);
```

Under this new mode, Babel would try to preserve the original formatting as much as possible:
```js
var x = function () {
  return (
    1 + 2
  ) * 3;
};

console.log(x);

for (var i = 0; i<3; i+=1)
  console.log(i);
```

# Motivation

While we think of Babel's output as basically "bytecode for the browser", and not meant to be read/modified by humans, that is in practice not the reality.

There are two main reasons why as a user you may want Babel to better preserve as much code style information as possible when transforming the code:

## Code migration tools

Babel can be used as a tool to write "codemods": you write a pugin to transform your code, for example to migrate across versions of your library, and run it on your _source_ file overwriting them. See, for example:
- https://github.com/kentcdodds/babel-codemod-example
- https://assets.ctfassets.net/nn534z2fqr9f/3bytbtRUOWLkWugqGgn174/18fa7bf319ac168a5aa7644747a30bfb/Babel__A_refactoring_tool.pdf
- https://github.com/codemod-js/codemod

These tools however face smoe challanges when it comes to preserve the output format:
1. they can assume that the input code was formatted with Prettier, and re-format the output code with Prettier;
2. they can use Babel together with [recast](https://github.com/benjamn/recast), but this only works when Recast is up-to-date with whatever Babel AST nodes are being used.
3. use Babel to find the locations where to change the code, and then do manual string slicing rather than AST transforms

## Debugging the compiled code

The more similar the compiled code is to the originally authored one, the easier it is to debug it. Other than step-by-step debugging, having accurate line _and column_ numbers ensures that stack traces point to the appropriate locations.

One alternative solution to this is to use source maps, rather than preserving the code formatting, but this also has its drawbacks:
- the quality of source maps is not always optimal (including those generated by Babel), leasing to potentially confusing 
- given that source maps can hide the code that is being executed, some companies' security policies recommend not using them and instead always checking the code that is actually running

# Detailed design

## API

We currently have two options to preserve some formatting in the output code:
- `retainLines`, that preserves line numbers
- `retainFunctionParens`, that preserves parentheses around function expressions (because some engines use them as a hint to eagerly compile the function body)

To implement this RFC, we need two new options:
- `retainParens`, to preserve all parentheses from the input code.
- `retainColumns`, to preserve the _column_ of the tokens other than their lines.

## Implementation

A guiding principle is that this new option should not require all the transforms, and most of the code printing logic, to be aware of it. Instead, it should be able to automatically match the locations of the generated code to those in the original code. It's possible that some plugins would need some ad-hoc handling, similarly to how sometimes we need to explicitly change some locations to improve source maps, but that must be the exception and not the rule.

While `@babel/generator` doesn't internally build a list of tokens, it concatenates string and every piece is de facto one JavaScript token. Those can be seen in the various [`.word("...")`](https://github.com/babel/babel/blob/8a8785984fddeed53934b5e23d0990050a196523/packages/babel-generator/src/generators/statements.ts#L262) and [`.token("...")`](https://github.com/babel/babel/blob/8a8785984fddeed53934b5e23d0990050a196523/packages/babel-generator/src/generators/statements.ts#L322) calls through the code.

The [core of the code generator](https://github.com/babel/babel/blob/c36fa6a9299df95b801695736f78ac87211f009f/packages/babel-generator/src/printer.ts) maintains a stack of nodes that are being printed, so whenever one of those `.word`/`.token` calls happen, the [core of the code generator](https://github.com/babel/babel/blob/c36fa6a9299df95b801695736f78ac87211f009f/packages/babel-generator/src/printer.ts) happens we know which code we are printing. We can thus traverse the list of input tokens, and if there is any token in the input list that:
1. matches the token that we are printing
2. is a _direct descendant_ of the node we are printing

we can use its location.

This alrogithm will probably require some caching to avoid traversing the list of tokens multiple times, as well as some optimized search based on the fact that most tokens are going to be printed after the same token that preceeded them in the input stream.

The (2) requirement is needed so that when, for example, printing the outer `BinaryExpression` in `(1 + 2) + 3`, we correctly associate the `+` token to print with the second `+` token in the input and not with the first one.

This strategy works because most nodes directly contain at most one occurrence of the same token. There are a few exceptions, that will probably need special handling:
- `,` in lists (function arguments, array literals, object literals, import attributes)
- `;` in the head of `for` loops

## Testing

We should have two types of tests: both with and withuot transformations applied.

For the first case, we can just re-use all the `input.js` files in the generators test and verify that `generate(parse(input)) === input`.

For transformations-related tests, such as testing what happens when converting `const` to `var`, we should probably keep the tests in the transform plugin packages.

# Drawbacks

This change can have a significant performance impact, since for every part of the output code `@babel/generator` has to find the original corresponding token in the tokens list to match its position.

We must make sure that the logic is self-contained enough to not affect performance of the classic code printer.

# Alternatives

It would be possible to implement this as a separate code generator, that users can specify through the [`generatorOverride`](https://github.com/babel/babel/blob/8a8785984fddeed53934b5e23d0990050a196523/packages/babel-core/src/config/plugin.ts#L13) option. However, having it as a separate fork will have a much higher maintainance cost.

# Adoption strategy

This is opt-in, and it will never replace the default behavior. Users that need it can simply turn on the new mode.

# How we teach this

This isn't a particularly complex concept to understand, so simply documenting the new option would be enough.

Once that is done, it'd be good to have examples of how it can be used to refactor code.

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

**What is the exact interaction between `retainLines` and `retainColumns`?**

It would be useful to be able to use `retainColumns` without also having to use `retainLines`, but with it still trying to preserve relative column changes.

For example, given this input code:
```js
let a =
    
    1 + 2;
```
and a transform that simply adds `"use strict";` at the beginning, there are two possible output. One that preserves line numbers and column numbers as much as possible:
```js
"use strict";let a=

    1 + 2;
```
and one that preserves _formatting_ as much as possible, even if the line numbers don't match anymore:
```js
"use strict";
let a =
    
    1 + 2;
```

Maybe instead of `retainLines`/`retainColumns` we should have `retainLines`/`preserveFormat`, where for now `preserveFormat` requires `retainLines` (giving the first output) but in the future could work independently (giving the second output). `preserveFormat` would also imply `retainParens`.

What approach is the best?

**What to do about `<!--` and `-->` comments?**

Given that `<!--` and `-->` are interpreted as comments in some environments, Babel always enforces a space between them: `<! --` and `-- >`. Here the choice is between preserving the exact code formatting, or preserving the code meaning (i.e. making sure that they are not interpreted as comments).

I'd erro towards the side of safety, but if we decide to go for the other behavior I'd want the safe behavior to still be presend behind some option/plugin.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

This is a common feature request:

- [#16301](https://github.com/babel/babel/issues/16301) — _[Bug]: The source code is indented to 4 Spaces, babel translated into 2_
- [#10674](https://github.com/babel/babel/issues/10674) — _@babel/generator output code different with input code
- [#8974](https://github.com/babel/babel/issues/8974) — _@babel/generator is removing the parentheses_
-  [#497](https://github.com/babel/babel/issues/497) — _Retaining whitespace/pretty printing_
