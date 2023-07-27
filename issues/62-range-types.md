Range types are types that represent ranges of integers.

# Discussion
Range types are a narrowing of the `int` type to a range of integers. Range types contain values in a given range of integers, e.g., an integer between 3 and 6.

If *a* and *b* are integer literals and *a &lt; b*, then the syntax `‹a› .. ‹b›` represents a type containing all the integers *x* such that *a &le; x &lt; b*. Rather, if *a &gt; b*, then `‹a› .. ‹b›` represents integers *x* such that *a &ge; x &gt; b*. The range `‹a› .. ‹b›` is called a **half-closed range**, because it includes *a* but excludes *b*. In standard mathematical notation, the half-closed range `‹a› .. ‹b›` would be expressed as the interval *[a, b)* or *(b, a]* (depending on how *a* compares to *b*).

The syntax `‹a› ... ‹b›` is called a **closed range** because it includes *b*: It represents all the integers *x* such that either *a &le; x &le; b* or *a &ge; x &ge; b*. In standard mathematical notation, the closed range `‹a› ... ‹b›` would be expressed as the interval *[a, b]* or *[b, a]* (depending on how *a* compares to *b*).

Range types are implemented as a type union of integer unit types. For example, the range type `3 .. 6` represents the union `3 | 4 | 5`. Decreasing ranges work similarly: The range type `6 .. 3` is `6 | 5 | 4`. In general, notice that `‹a› .. ‹b›` and `‹b› .. ‹a›` are not equivalent.

The range `3 ... 6` represents the type union `3 .. 6 | 6` (that is, the union `3 | 4 | 5 | 6`) and the range `6 ... 3` is `6 | 5 | 4 | 3`. Because the type union operation is commutative and associative, types `‹a› ... ‹b›` and `‹b› ... ‹a›` are the same.

Range types are normal type expressions. Any range type is a subtype of `int`.
```cp
type T = 3 .. 6;
let i: T   = 5;
let j: T   = 9; %> TypeError
let k: int = i;
```

Some more examples:
```cp
type A =  3 ..   3; % `never`
type B =  3 ..   4; % `3`
type C =  4 ..   3; % `4`
type D =  3 ...  3; % `3`
type E =  3 ...  4; % `3 | 4`
type F =  4 ...  3; % `4 | 3`
type G = -1 ..   3; % `-1 | 0 | 1 | 2`
type H =  1 ..  -3; % `1 | 0 | -1 | -2 `
```

Overlapping ranges can be intersected and unioned:
```cp
type V = 3 .. 7 & 5 .. 9; % equivalent to `5 .. 7`
type W = 3 .. 7 | 5 .. 9; % equivalent to `3 .. 9`
```

Ranges are stronger than type operations. `2 & 3 .. 6` is interpreted as `2 & (3 .. 6)`.

The variables *a* and *b* mentioned above are only meta-variables; the syntax of ranges requires *integer literals*.
```cp
let a: int = 3;
let b: int = 6;
let c: a .. b = 5; %> ParseError
```

We can use integers with different bases and numeric separators, and we can even mix bases.
```cp
let c: \x1_0000 ... \x10_ffff = \x1_ffff; % an integer between 65_536 and 1_114_111, set to 131_071
let d: \o5      ... \zt       = 10;       % an integer between      5 and        20, set to      10
```

Note that there should be whitespace before the symbols `..` and `...` in ranges, as omitting the whitespace sometimes poses a lexical ambiguity. For example, a range written as `3...6` would be lexed not as `3 ... 6`, but as `3. .. 6` (a parse error), since float tokens may omit their fractional part. (Technically, `3 ...6` is well-formed, but programmers might prefer `3 ... 6` aesthetically.) This whitespace may be omitted when the left-hand integer uses a radix, e.g. `\d3...\d6` is well-formed, since `\d3.` cannot be lexed as a float.

# Specification

## Lexicon
```diff
Punctuator :::=
	// grouping
+		| ".." | "..."
;
```

## TokenWorth
```diff
TokenWorth(Punctuator  :::= ",")   -> RealNumber := \x04;
+TokenWorth(Punctuator :::= "..")  -> RealNumber := \x05;
+TokenWorth(Punctuator :::= "...") -> RealNumber := \x06;
-TokenWorth(Punctuator :::= "|->") -> RealNumber := \x05;
+TokenWorth(Punctuator :::= "|->") -> RealNumber := \x07;
# and so on…
```

## Syntax
```diff
+RangeLiteral
+	::= INTEGER (".." | "...") INTEGER;

TypeUnit ::=
	| IDENTIFIER
	| PrimitiveLiteral
+	| RangeLiteral
	| TypeKeyword
	| "(" Type ")"
;
```

## Decorate
```diff
+Decorate(RangeLiteral ::= INTEGER__0 ".." INTEGER__1) -> SemanticType
+	:= RangeHalfClosed(Integer(TokenWorth(INTEGER__0)), Integer(TokenWorth(INTEGER__1)));

+Decorate(RangeLiteral ::= INTEGER__0 "..." INTEGER__1) -> SemanticTypeOperation
+	:= (SemanticTypeOperation[operator=OR]
+		RangeHalfClosed(Integer(TokenWorth(INTEGER__0)), Integer(TokenWorth(INTEGER__1)))
+		(SemanticTypeConstant[value=ToType(Integer(TokenWorth(INTEGER__1)))])
+	);

+Decorate(TypeUnit ::= RangeLiteral) -> SemanticType
+	:= Decorate(RangeLiteral);
```

### RangeHalfClosed
```
SemanticType RangeHalfClosed(Integer a, Integer b) :=
	1. *If* `b` equals `a`:
		1. *Return:* `(SemanticTypeConstant[value=Never])`.
	2. *If* `b` is less than `a`:
		1. *Return:* `RangeHalfClosed(b + 1, a + 1)`.
	3. *Assert:* `a` is less than `b`.
	4. *Return:* `(SemanticTypeOperation[operator=OR]
		RangeHalfClosed(a, b - 1)
		(SemanticTypeConstant[value=ToType(b - 1)])
	)`.
;
```
