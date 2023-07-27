Tuple segment accessors are an extension of the bracket accessor (#25) and produces a tuple of items.

# Discussion
The examples in this section assume the following:
```cp
let tup: [str, str, str, str, str, str, str] = [a, b, c, d, e, f, g]; % assume these variables are strings
```

The bracket accessor of a tuple may include two expressions to produce a segment of entries from the tuple. The syntax `tuple.[‹a› .. ‹b›]` (where ‹a› and ‹b› are metavariables repesenting expressions) returns a read-only segment of the tuple from indices ‹a› to ‹b›, *excluding* ‹b›. The syntax is inspired by range types (#62).
```cp
tup.[2 .. 5] == [c, d, e]; % indices 2, 3, 4
```

Produced entries will always be in increasing index order, and never reversed. If ‹b› is less than or equal to ‹a›, *and both are positive*, then `tuple.[‹a› .. ‹b›]` produces an empty tuple.
```cp
tup.[3 .. 3] == [];
tup.[5 .. 2] == [];
```

As with regular dot and bracket accessors, negative indices count leftward from the end of the tuple (with `-1` being the last index). Negative indices are converted to postive indices *before* the range is computed.
```cp
tup.[ 1 .. -2] %% tup.[1 .. 5] %% == [b, c, d, e];
tup.[-5 .. -3] %% tup.[2 .. 4] %% == [c, d];
tup.[-3 .. -6] %% tup.[4 .. 1] %% == [];
tup.[-3 ..  1] %% tup.[4 .. 1] %% == [];
tup.[-1 ..  3] %% tup.[6 .. 3] %% == [];
```

To produce a segment from a given index to the end of the tuple, use `0` or the tuple’s count as the second index:
```cp
tup.[3 .. 0] == [d, e, f, g]; % indices 3, 4, ... to the end, including the last index
tup.[3 .. 7] == [d, e, f, g]; % indices 3, 4, 5, 6 (the last index)
```

To produce a segment from the start of the tuple to a given index, use `0` as the first index:
```cp
tup.[0 ..  3] == [a, b, c];    % indices 0, 1, 2
tup.[0 .. -3] == [a, b, c, d]; % `tup.[0 .. 4]`
```

Use three dots `...` to *include* the second index.
```cp
tup.[ 3 ...  5] == [d, e, f];       % indices 3, 4, 5
tup.[-5 ... -3] == [c, d, e];       % indices 2, 3, 4
tup.[3  ...  0] == [d, e, f, g];    % indices 3, 4, ... to the end, but does not loop around (same as `3 .. 0`)
tup.[0  ... -3] == [a, b, c, d ,e]; % `tup.[0 ... 4]`
```

In all of the examples above, we used integer literals as indices. However, any expression, even if it’s dynamic, is allowed. Because `..` and `...` have a high precedence, we must enclose any operations in parentheses.
```cp
let a: int = 3;
let b: int = 6;
tup.[(2 + 3) .. -1]       == [f];          % `tup.[5 .. 6]`
tup.[3 .. (+tup)]         == [d, e, f, g]; % `tup.[3 .. 7]`, where `+tup` is its count, 7
tup.[(a - 1) ... (b - 1)] == [c, d, e, f]; % `tup.[2 ... 5]`
```

# Specification

## Lexicon
```diff
Punctuator :::=
	// grouping
+		| ".." | "..."
;
```

## Syntax
```diff
+Range
+	::= ExpressionCompound (".." | "...") ExpressionCompound;

PropertyAccess
-	::= "." (INTEGER | Word | "[" Expression           "]");
+	::= "." (INTEGER | Word | "[" (Expression | Range) "]");

ExpressionCompound ::=
	| ExpressionUnit
	| ExpressionCompound PropertyAccess
;
```

## Semantics
```diff
+SemanticRange
+	::= SemanticExpression SemanticExpression;

SemanticAccess
-	::= SemanticExpression (SemanticIndex | SemanticKey | SemanticExpression                );
+	::= SemanticExpression (SemanticIndex | SemanticKey | SemanticExpression | SemanticRange);
```

## Decorate
```diff
+Decorate(Range ::= ExpressionCompound__0 (".." | "...") ExpressionCompound__1) -> SemanticRange
+	:= (SemanticRange
+		Decorate(ExpressionCompound__0)
+		Decorate(ExpressionCompound__1)
+	);

Decorate(PropertyAccess ::= "." "[" Expression "]") -> SemanticExpression
	:= Decorate(Expression);
+Decorate(PropertyAccess ::= "." "[" Range "]") -> SemanticRange
+	:= Decorate(Range);

Decorate(ExpressionCompound ::= ExpressionCompound PropertyAccess) -> SemanticAccess
	:= (SemanticAccess
		Decorate(ExpressionCompound)
		Decorate(PropertyAccess)
	);
```

# Workaround
If this feature is not implemented, a sufficient workround is to construct a new tuple with the desired entries.
```cp
% tup.[2 .. 5];
[
	tup.[2],
	tup.[3],
	tup.[4],
];
```
