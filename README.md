# Extended Numeric Literals

Stage 1

Champion: Daniel Ehrenberg

## Motivation

### Generalizing BigInt

Currently, JavaScript contains just one numeric type, [Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number). A second type, [BigInt](https://github.com/tc39/proposal-bigint), is proposed. To differentiate the literal syntaxes, BigInts are written ending with an `n`, e.g., `19824359823509831298352352n`. In the case of BigInt, this is a one-off change to JavaScript grammar.

Other numeric types which may be added, either in JavaScript library code or eventually as a built-in:
- IEEE 754-2008 64-bit decimals
- Arbitrary-precision decimals
- Rationals
- Complex numbers

One side to all of these is operator overloading, and another side is the literal syntax. In some small code samples, it seems like using extended literals, together with methods for arithmetic operators, might not be such a bad middle point, until more is worked out.

### CSS Typed Object Model

In the [CSS Typed Object Model](https://drafts.css-houdini.org/css-typed-om/#numeric-factory), there are objects which represent lengths in pixels, inches, and several other units. The current syntax for creating such an instance is `CSS.px(10)`. With this proposal, the syntax could instead be just like inside of CSS itself, as `10_px`. This is another case that would benefit from operator overloading, but also be useful without it.

Example:

```js
import { numeric__px } from "@std/css";

document.querySelector("#foo").style.fontSize = 3~px;
```

## Proposed syntax

Numeric literals have the syntax of a Number followed by a `~`, followed by numeric literal suffix. The numeric literal suffix uses the `IdentifierName` grammar, similar to a property key which comes after `.`.

```
PrimaryExpression[Yield, Await] :
  ...
  ExtendedNumericLiteral

ExtendedNumericLiteral ::
  NumericLiteral `~` IdentifierName
```

Whitespace is not permitted either before or after the `~` character; this restriction is encoded in the grammar by being part of the lexical, rather than syntactic, grammar (:: not :).

## Semantics

Similarly to template literals, extended numeric literals desugar into passing an object literal as an argument to a function. The object has two own properties:
- `string`: The literal source text preceding the `~`.
- `number`: `string` interpreted as a literal Number. The parsed form is important for users like CSS Typed OM, which needs to avoid re-parsing for performance reasons.

Example:

`3~px` desugars into `numeric__px(Object.freeze({number: 3, string, "3"}))`

### Caching

The object which is passed into the function is cached such that multiple executions of the same code will have the same object passed into the extended literal function. This can be useful for literals which require an expensive calculation to parse: the object can be used as a key in a WeakMap associating it with a pre-calculated value.

Analogous to the semantics recently adopted template string literals ([PR](https://github.com/tc39/ecma262/pull/890)), extended numeric literal objects are cached by source position, rather than the contents of the numeric literal.
