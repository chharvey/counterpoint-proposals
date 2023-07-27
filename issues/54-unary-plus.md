Re-purpose or overload unary prefix operator `+` to compute “length”/“size” for strings and collections.

# Discussion
```
`+` <int | float>
```
The **mathematical affirmation** operator, `+`, is still a no-op for number types.

## Overload: Collection Size
```
`+` <str | Tuple | Record | Mapping>
```
The **size** operator, `+`, computes the size of a collection and returns a non-negative Integer:

- the number of characters in a string
- the number of items in a tuple
- the number of properties in a record
- the number of cases in a mapping

Example:
```
+'hello world' == 11;
+[null, true, false] == 3;
```

# Spec

## Schema
```diff
-SemanticOperation[operator: NEG]
+SemanticOperation[operator: AFF | NEG]
	::= SemanticExpression[type: Number];
```

## Decorate
```diff
-Decorate(ExpressionUnarySymbol ::= "+" ExpressionUnarySymbol) -> SemanticExpression
-	:= Decorate(ExpressionUnarySymbol);
+Decorate(ExpressionUnarySymbol ::= "+" ExpressionUnarySymbol) -> SemanticOperation
+	:= (SemanticOperation[operator=AFF]
+		Decorate(ExpressionUnarySymbol)
+	);
```

## TypeOf
```
Type! TypeOfUnassessedOperation(SemanticOperation[operator: AFF] expr) :=
	1. *Assert:* `expr.children.count` is 1.
	2. *Let* `t` be *Unwrap:* `TypeOf(expr.children.0)`.
	3. *If* `t` is a subtype of `Union(Integer, Float)`:
		1. *Return:* `t`.
	4. *If* `t` is a subtype of `String`:
		1. *Let* `n` be the number of code units in `t`.
	5. *If* `t` is a subtype of `Tuple`:
		1. *Let* `n` be the number of type items in `t`.
	6. *If* `t` is a subtype of `Record`:
		1. *Let* `n` be the number of type properties in `t`.
	7. *If* `t` is a subtype of `Mapping`:
		1. *Let* `n` be the number of type cases in `t`.
	8. *If* `n` is set:
		1. *Return:* `Integer(n)`.
	8. *If* `t` is a subtype of `Union(String, Tuple, Record, Mapping)`:
		1. *Return:* `Integer`.
	9. *Throw:* a new TypeError01.
;
```

## Assess
```
Number! Assess(SemanticOperation[operator: AFF] expr) :=
	1. *Assert:* `expr.children.count` is 1.
	2. *Let* `operand` be *Unwrap:* `Assess(expr.children.0)`.
	3. *If* `operand` is of type `Number`:
		1. *Return:* `operand`.
	4. *If* `operand` is of type `String`:
		1. *Return:* the number of code units in `operand`.
	5. *If* `operand` is of type `Tuple`:
		1. *Return:* the number of items in `operand`.
	6. *If* `operand` is of type `Record`:
		1. *Return:* the number of properties in `operand`.
	7. *Assert:* `operand` is of type `Mapping`.
	8. *Return:* the number of cases in `operand`.
;
```

## Build
TODO

## Evaluate
TODO

# TODO
- [ ] remove negated semantic operators `NLT`, `NGT`, `ISNT`, `NID`, etc.
- [ ] add new semamtic operators `UNION`, `INTSEC`
