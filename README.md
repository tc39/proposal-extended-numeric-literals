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

Whitespace is not permitted between the numeric part and the identifier; this is encoded in the grammar by being part of the lexical, rather than syntactic, grammar (:: not :).

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

The downside here is that such concatenation of names is not typical in JS. When defining and importing literals, the syntax is highly unintuitive--it plainly leaks into everything.

The issue can be mitigated by introducing special forms for defining such literals. For example, `function literal i() { ... }`, and `import {literal i} from "imaginary.js"`. However, then you're still left with a weird property that, if you type the "mangled" name by accident, the abstraction will leak.

#### Separate namespace

To prevent the leakiness of the abstraction, mangled names could be made untypable. For example, as a space is not permitted in an identifier in JS, the "lexically scoped name" could be `literal i`. For example: It could be defined by forms like `function literal i() {}`, `let literal i = ...`, `import {literal i}`, but for reading, it would only be accessible when used in suffix position.

The effect of this would be to introduce a separate, parallel namespace for literals. Such a namespace exists for private names in the [class fields proposal](https://github.com/tc39/proposal-class-fields). In both this proposal and that one, this is not a general namespace which overlaps on ordinary things, but instead a distinct syntax which selects it. Therefore, the hope is that it would not lead to as much complexity as earlier ES4 proposals, or the current state of C++.

Thanks to Mark Miller for making this suggestion.

#### Property of an object

A simpler approach might be to instead use an object for namespacing. There could be a global `%Literals%` object which would have properties like `i`, which are the functions here.

There are several downsides, though:
- Literals are now based on mutating a globally shared object, rather than defining a lexically scoped value; this could feel like a step in the wrong direction.
- We would need to integrate the Literals object with any Realm API, so a realm can replace it.
- For userspace realm systems which want to prevent communication channels, like SES, the only short-term alternative would be to disable defining new literals by freezing the object.

## Possible extensions

### Extensible Map/Array literals

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

Extensible map/array literals may be better candidates for using plain old lexical scope than numeric literals, as a result of the whole literal just being bigger.
