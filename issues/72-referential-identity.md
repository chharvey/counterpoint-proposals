This issue changes the “referential identity” (“strict equality”) operator from `is` to `===` and its negation from `isnt` to `!==`.

The referential identity operator and its negation will become `===` and `!==` respectively. The tokens `is` and `isnt` will be kept allowed in the syntax, but as comparison operators rather than as equality operators. For now, they will result in a compile-time TypeError, but are reserved for later use (object ‘instanceof’ when we get classes).

As a review: **Referential identity** means two operands point to the exact same object (if they are references), or they are represented exactly the same bitwise (if they are primitives including strings). Thus `-0.0` and `0.0` are not identical, nor are `42` and `42.0`, nor are `[42]` and `[42]`.

```cp
-0.0 === 0.0; %== false
-0.0 !== 0.0; %== true
42 === 42.0; %== false
42 !== 42.0; %== true

let o1: obj = [42, 420];

[42, 420] === [42, 420]; %== false
[42, 420] !== [42, 420]; %== true
o1 === [42, 420]; %== false
o1 !== [42, 420]; %== true
o1 === o1; %== true
o1 !== o1; %== false

let o2: obj = [a= 42, b= 420];

[a= 42, b= 420] === [a= 42, b= 420]; %== false
[a= 42, b= 420] !== [a= 42, b= 420]; %== true
o2 === [a= 42, b= 420]; %== false
o2 !== [a= 42, b= 420]; %== true
o2 === o2; %== true
o2 !== o2; %== false
```

As a review: **Equality** of objects depends on their type: null, boolean, and string values are equal iff they are identical; numeric primitives are equal iff they have the same mathematical value (`-0.0` and `0.0` are equal, as are `42` and `42.0`); and collection objects are equal iff they have equal entries by index or by key (e.g. `[42]` and `[42]`).

# Specification

## Lexicon
```diff
Punctuator :::=
	// binary
-		|                 "==" | "!="
+		| "===" | "!==" | "==" | "!="
;
```

## TokenWorth
```diff
-TokenWorth(Punctuator :::= "==")  -> RealNumber := \x13;
-TokenWorth(Punctuator :::= "!=")  -> RealNumber := \x14;
+TokenWorth(Punctuator :::= "===") -> RealNumber := \x13;
+TokenWorth(Punctuator :::= "!==") -> RealNumber := \x14;
+TokenWorth(Punctuator :::= "==")  -> RealNumber := \x15;
+TokenWorth(Punctuator :::= "!=")  -> RealNumber := \x16;
# and so on
```

## Syntax
```diff
ExpressionComparative
-	::= (ExpressionComparative ("<" | ">" | "<=" | ">=" | "!<" | "!>"))?                 ExpressionAdditive;
+	::= (ExpressionComparative ("<" | ">" | "<=" | ">=" | "!<" | "!>" | "is" | "isnt"))? ExpressionAdditive;

ExpressionEquality
-	::= (ExpressionEquality ("is" | "isnt" | "==" | "!="))? ExpressionComparative;
+	::= (ExpressionEquality ("===" | "!==" | "==" | "!="))? ExpressionComparative;
```

## Semantics
```diff
-SemanticOperation[operator: IS | EQ]
+SemanticOperation[operator: IDEN | EQ]
	::= SemanticExpression[type: Object] SemanticExpression[type: Object];
```

## Decorate
```diff
+Decorate(ExpressionComparative ::= ExpressionComparative "is" ExpressionAdditive) -> SemanticOperation
+	:= (SemanticOperation[operator=IS]
+		Decorate(ExpressionComparative)
+		Decorate(ExpressionAdditive)
+	);
+Decorate(ExpressionComparative ::= ExpressionComparative "isnt" ExpressionAdditive) -> SemanticOperation
+	:= (SemanticOperation[operator=NOT]
+		(SemanticOperation[operator=IS]
+			Decorate(ExpressionComparative)
+			Decorate(ExpressionAdditive)
+		)
+	);

-Decorate(ExpressionEquality ::= ExpressionEquality "is" ExpressionComparative) -> SemanticOperation
+Decorate(ExpressionEquality ::= ExpressionEquality "===" ExpressionComparative) -> SemanticOperation
-	:= (SemanticOperation[operator=IS]
+	:= (SemanticOperation[operator=ID]
		Decorate(ExpressionEquality)
		Decorate(ExpressionComparative)
	);
-Decorate(ExpressionEquality ::= ExpressionEquality "isnt" ExpressionComparative) -> SemanticOperation
+Decorate(ExpressionEquality ::= ExpressionEquality "!==" ExpressionComparative) -> SemanticOperation
	:= (SemanticOperation[operator=NOT]
-		(SemanticOperation[operator=IS]
+		(SemanticOperation[operator=ID]
			Decorate(ExpressionEquality)
			Decorate(ExpressionComparative)
		)
	);
```

## TypeOf
```diff
+Type! TypeOfUnassessedOperation(SemanticOperation[operator: IS] expr) :=
+	1. *Throw:* new TypeError "Operator not yet supported."
+;

-Type! TypeOfUnassessedOperation(SemanticOperation[operator: IS | EQ] expr) :=
+Type! TypeOfUnassessedOperation(SemanticOperation[operator: ID | EQ] expr) :=
	1. *Assert:* `expr.children.count` is 2.
	2. *Let* `t0` be *Unwrap:* `TypeOf(expr.children.0)`.
	3. *Let* `t1` be *Unwrap:* `TypeOf(expr.children.1)`.
	4. *If* `t0` is a subtype of `Number` *and* `t1` is a subtype of `Number`:
		1. *If* `t0` is a subtype of `Integer` *or* `t1` is a subtype of `Integer`:
			1. *If* `t0` is a subtype of `Float` *or* `t1` is a subtype of `Float`:
				1. *If* `operator` is `IS`:
					1. *Return:* `ToType(false)`.
		2. *Return:* `Boolean`.
	5. *If* `t0` and `t1` are disjoint:
		1. *Return:* `ToType(false)`.
	6. *Return:* `Boolean`.
;
```

## Assess
```diff
+Boolean! Assess(SemanticOperation[operator: IS] expr) :=
+	1. *Throw:* new TypeError "Operator not yet supported."
+;

-Boolean! Assess(SemanticOperation[operator: IS] expr) :=
+Boolean! Assess(SemanticOperation[operator: ID] expr) :=
	1. *Assert:* `expr.children.count` is 2.
	2. *Let* `operand0` be *Unwrap:* `Assess(expr.children.0)`.
	3. *Let* `operand1` be *Unwrap:* `Assess(expr.children.1)`.
	4. *Assert*: Neither `operand0` nor `operand1` is `void`.
	5. *Return:* `Identical(operand0, operand1)`.
;
```

## Build
```diff
+Sequence<Instruction> BuildExpression(SemanticOperation[operator: IS] expr) :=
+	1. *Throw:* new TypeError "Operator not yet supported."
+;

-Sequence<Instruction> BuildExpression(SemanticOperation[operator: EXP | MUL | DIV | ADD | LT | GT | LE | GE | IS | EQ] expr) :=
+Sequence<Instruction> BuildExpression(SemanticOperation[operator: EXP | MUL | DIV | ADD | LT | GT | LE | GE | ID | EQ] expr) :=
	1. *Let* `builds` be *UnwrapAffirm:* `PrebuildSemanticOperationBinary(expr)`.
	2. *Return:* [
		...builds.0,
		...builds.1,
		"`expr.operator`",
	].
;
```
