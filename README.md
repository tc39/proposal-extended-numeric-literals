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
let { _px } = CSS;

document.querySelector("#foo").style.fontSize = 3_px;
```

## Proposed syntax

Numeric literals have the syntax of a Number followed by a numeric literal suffix. The suffix must begin with an underscore, followed by the remainder of what would form an Identifier.

```
PrimaryExpression[Yield, Await] :
  ...
  ExtendedNumericLiteral

ExtendedNumericLiteral ::
  NumericLiteral `_` IdentifierPart
```

Whitespace is not permitted between the numeric part and the identifier; this is encoded in the grammar by being part of the lexical, rather than syntactic, grammar (:: not :).

## Semantics

Similarly to template literals, extended numeric literals desugar into passing an object literal as an argument to a function. The object has two own properties:
- `string`: The literal source text preceding the `_`.
- `number`: `string` interpreted as a literal Number. The parsed form is important for users like CSS Typed OM, which needs to avoid re-parsing for performance reasons.

Example:

`3_px` desugars into `_px(Object.freeze({number: 3, string: "3"}))`

### Caching

The object which is passed into the function is cached such that multiple executions of the same code will have the same object passed into the extended literal function. This can be useful for literals which require an expensive calculation to parse: the object can be used as a key in a WeakMap associating it with a pre-calculated value.

New semantics are under consideration for template string literals ([PR](https://github.com/tc39/ecma262/pull/890)) and would make sense for numeric literals as well. While under development, this proposal matches the main specification's logic, caching by the string value of the NumericLiteral rather than source position.

## Future work

### Extended Map/Array literals

The syntactic convenience of JS object literals is one reason why they are still in wide use even when Maps are available. One part of this is property access, but another part is literals. The issue has been [raised on es-discuss](https://esdiscuss.org/topic/map-literal), but there is currently no proposal for a change here.

One option would be to allow similar literal prefixes or suffixes on array and object literals, as follows:

```js
// If we choose prefix
// Array literals don't work
mymap{["foo"]: 1, [[bar]]: 2}

// If we choose suffix
[1, 2, 3]mylist
{["foo"]: 1, [[bar]]: 2}mymap
```

In the earlier es-discuss thread, a `#` was used between the name of the literal and the main literal itself; this seems unnecessary syntactically as long as no line break is permitted (but we could still do it for aesthetic reasons).

Extended map or array literals seem to face a different set of syntactic constraints and a larger API surface. It seems that additions here can go type-by-type, as things started with template literals, and this proposal adds numeric literals; there is no dependency between them, and no difficulty in numeric literals following the patterns of template literals. For that reason, they are left for a follow-on proposal.
