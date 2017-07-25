# Extensible Numeric Literals

Daniel Ehrenberg

## Motivation

### Generalizing BigInt

Currently, JavaScript contains just one numeric type, [Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number). A second type, [BigInt](https://github.com/tc39/proposal-bigint), is proposed. To differentiate the literal syntaxes, BigInts are written ending with an `n`, e.g., `19824359823509831298352352n`. In the case of BigInt, this is a one-off change to JavaScript grammar.

Other numeric types which may be added, either in JavaScript library code or eventually as a built-in:
- IEEE 754-2008 64-bit decimals
- Arbitrary-precision decimals
- Rationals
- Complex numbers

One side to all of these is operator overloading, and another side is the literal syntax. In some small code samples, it seems like using extensible literals, together with methods for arithmetic operators, might not be such a bad middle point, until more is worked out.

### CSS Typed Object Model

In the [CSS Typed Object Model](https://drafts.css-houdini.org/css-typed-om/#numeric-factory), there are objects which represent lengths in pixels, inches, and several other units. The current syntax for creating such an instance is `CSS.px(10)`. With this proposal, the syntax could instead be just like inside of CSS itself, as `10px`. This is another case that would benefit from operator overloading, but also be useful without it.

## Proposed syntax

Numeric literals have the syntax of a Number followed by a numeric literal suffix. This suffix may be any Identifier which does not have a name that would cause ambiguities. BigInts have an Early Error for using a decimal or exponential form, but user-defined literals will only be able to trigger runtime errors for these sorts of cases.

```
NumericLiteral ::
  NumericLiteralBase
  NumericLiteralBase NumericLiteralSuffix
  LegacyOctalIntegerLiteral

NumericLiteralBase ::
  DecimalLiteral
  BinaryIntegerLiteral
  OctalIntegerLiteral
  HexIntegerLiteral
  
NumericLiteralSuffix ::
  n
  Identifier which does not start with xXoObB and is either a single character, or contains a following character which is not digits 0-9,a-f,A-F; the Identifier may not be n
```

The restriction in the allowed identifiers is done to disambiguate cases like `0xf`--this was previously specified to be a hexadecimal literal Number, but could be mistaken for `0` followed by the extensible numeric literal suffix `xf`, as `xf` is a perfectly fine Identifier.

## Semantics

Extensible literals are simply functions which take two arguments:
- The string for the literal text preceding the literal suffix.
- That string parsed as a Number. The parsed form is important for users like CSS Typed OM, which needs to avoid re-parsing for performance reasons.

There is a possibility of including some sort of object for caching, as tagged template literals do, rather than or in addition to the pre-parsed number.

### Scoping variants

The tough part of the design of integer iterals is scoping. Where do you look up what tne numeric suffix means?

#### Purely lexical

The semantics here would be, look up the lexically scoped value, and call that as a function.

Take the case of an imaginary number suffix, which could naturally be named `i`. (TC39 didn't use `i` for BigInt because it just seems too perfect for imaginary numbers.) The problem here is, it is very likely for `i` to be shadowed lexically--it's commonly used as a loop variable! Using `j` for imaginary number literals instead doesn't mitigate the problem that much either.

#### Name mangling

One solution to the issue here is name mangling. Concatenate a `__literal_` prefix on the beginning. So, when defining the literal, write `function __literal_i() { ... }`, or when importing it, use `import {__literal_i} from "imaginary.js";`, and when using it, just use `1i`.

The downside here is that such concatenation of names is not typical in JS

#### Separate namespace

#### Property of an object

## Possible extensions

### Extensible Map/Array literals

The syntactic convenience of JS object literals is one reason why they are still in wide use even in 
