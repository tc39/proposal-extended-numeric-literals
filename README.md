# Numeric Literal Suffixes

![Stage 1](https://badges.aleen42.com/src/tc39_2.svg)

Stage 1

Champion: Daniel Ehrenberg

## Motivation

### Generalizing BigInt

JavaScript contains two numeric types, [Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number) and [BigInt](https://github.com/tc39/proposal-bigint). To differentiate the literal syntaxes, BigInts are written ending with an `n`, e.g., `19824359823509831298352352n`. In the case of BigInt, this is a one-off change to JavaScript grammar.

Other numeric types which may be added:
- IEEE 754-2008 128-bit decimals
- Arbitrary-precision decimals
- Rationals
- Complex numbers

The hope is that these could be added either built-in to the language *or* simply defined by JavaScript code, without using transpilers/tools. Part of the picture of generalizing built-in syntax is operator overloading; another part is custom numeric literal suffixes.

Example custom numeric type:

```js
import { Decimal } from "./my-decimal-package.mjs";

with suffix $ = Decimal.literal;
(2.1$).plus(3.2$) // ==> 5.3$
```

### CSS Typed Object Model

In the [CSS Typed Object Model](https://drafts.css-houdini.org/css-typed-om/#numeric-factory), there are objects which represent lengths in pixels, inches, and several other units. The current syntax for creating such an instance is `CSS.px(10)`. With this proposal, the syntax could instead be just like inside of CSS itself, as `10px`.

This is another case that would benefit from operator overloading, but also be useful without it.

Because CSS is built-in to the web environment, this example assumes that the environment makes it present in the default outer scope.

Example:

```js
document.querySelector("#foo").style.fontSize = 3px;
```

## Proposed syntax

Numeric literals have the syntax of a Number followed by a restricted kind of identifier. The identifier must begin with a character which is never present in a valid NumericLiteral.

```
NumericLiteral ::
  DecimalLiteral
  DecimalLiteral SuffixIdentifier
  NonDecimalIntegerLiteral
  NonDecimalIntegerLiteral SuffixIdentifier

SuffixIdentifier ::
  SuffixIdentifierStart IdentifierPart

SuffixIdentifierStart ::
  UnicodeIDStart but not one of HexDigit or o x O X
  $

LexicalDeclaration :
  LetOrConst BindingList
  SuffixDeclaration

SuffixDeclaration :
  with suffix SuffixIdentifier Initializer
```

Whitespace is not permitted between the |NumericLiteral| and the |SuffixIdentifier|. this restriction is encoded in the grammar by being part of the lexical, rather than syntactic, grammar (:: not :). Note that `_` is not a member of UnicodeIDStart.

Early errors for particular built-in numeric suffixes like `n` are still in place, but only trigger if that suffix is not shadowed by a user-defined numeric literal suffix. Non-built-in numeric literal suffixes can instead use runtime errors when they're evaluated.

## Scoping

Numeric literal suffixes live in a parallel lexical scope. Entries can be created in the scope with a `SuffixDeclaration`, of the form `with suffix s = expression`. Usages in numeric literals refer to one of those suffixes in the current or outer scope.

When importing a suffix from a module, the normal lexical scope is used. A single line in the importing module can be used to declare it in the current scope. A future proposal could allow direct exports/imports in the suffix namespace, but this proposal omits that capability, as it does not seem worth the cost of the added complexity.

## Semantics

User-defined numeric literal suffixes are designed in a way analogous to template literals: they are called with an object which is fixed based on the callsite. The object has two own properties:
- `string`: The literal source text preceding the suffix.
- `number`: `string` interpreted as a literal Number. The parsed form is important for users like CSS Typed OM, which needs to avoid re-parsing for performance reasons.

Here's a fully functioning polyfill for the hypothetical CSS `px` literal suffix:

```js
function pxSuffix({number}) {
  return CSS.px(number)
}

with suffix px = pxSuffix;

3px;
```

The last line desugars into:

```js
let template = Object.freeze({ number: 3, string: "3" });
pxSuffix(template);
```

### Caching

The object which is passed into the suffix function is cached such that multiple executions of the same code will have the same object passed into the extended literal function. This can be useful for literals which require an expensive calculation to parse: the object can be used as a key in a WeakMap associating it with a pre-calculated value.

Analogous to the semantics recently adopted template string literals ([PR](https://github.com/tc39/ecma262/pull/890)), extended numeric literal objects are cached by source position, rather than the contents of the numeric literal.

Using this caching logic, the `px` polyfill can be optimized to avoid creating a new object each time:

```js
const cache = new WeakMap();
function pxSuffix(obj) {
  if (cache.has(obj)) return cache.get(obj);
  const px = Object.freeze(CSS.px(obj.number));
  cache.set(obj, px);
  return px;
}
```

### Status

This proposal is at Stage 1 in TC39. The new version using a separate namespace for suffixes has not yet been presented to committee.
