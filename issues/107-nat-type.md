The `nat` (natural number) type contains unsigned integers.

Like signed integers, they are 64-bit, but range from *0* to *2^64 - 1* (instead of from *-(2^63)* to *2^63 - 1* like ints do). Unsigned integer values are encoded “as-is” in binary. Representations having an MSB of 1 (the leading bit) are interpreted as their numeric values, rather than as negative numbers like signed integers in two’s complement. There is no way to have a `nat` value less than 0. As with the integer type, there is only one representation for 0 in the natural number type.

The `nat` type is used for the number of entries in a countable and tuple/list indices. It can also be used anywhere non-negative integers are needed, such as in a factorial function. `nat` types and `int` types can be compared via the equality operators `==` and `!=` (without explicit casting needed). However, `nat` and `int` values, and `nat` and `float` values for that matter, cannot be combined together in arithmetic and comparator operations without explicit coercion first. For example, if `n` is a nat and `i` is an int, then the expression `n + i` would be invalid. The programmer must explicitly cast `n` into an int in order to validate the expression: `int n + i`.

## Syntax
The syntax for `nat` literals is a plus sign (`+`) followed by integer syntax. For example, `+42` represents the unsigned integer 42 whereas `42` represents the signed integer 42. Though their binary representations are identical, they are not type-compatible. (Note this is a breaking change, since previously `+42` would just be a no-op on the `int` value 42.) `float` values may still keep their unary `+` operator: `+42.0` is still a float.

Natural numbers can use the same non-decimal radices as integers. E.g. `\o+52 === +42`. See documentation for the Integer type for details.

### Lexicon
```diff
+Natural<Radix, Separator>
+	:::= "+" <Radix->DigitSequenceDec<?Separator> . <Radix+>WholeDigitsRadix<?Separator>

Integer<Radix, Separator>
-	:::= ("+" | "-")? <Radix->DigitSequenceDec<?Separator> . <Radix+>IntegerDigitsRadix<?Separator>;
+	:::= "-"?         <Radix->DigitSequenceDec<?Separator> . <Radix+>WholeDigitsRadix<?Separator>;

-IntegerDigitsRadix<Separator> :::=
+WholeDigitsRadix<Separator> :::=
	| "\b"  DigitSequenceBin<?Separator>
	| "\q"  DigitSequenceQua<?Separator>
	| "\s"  DigitSequenceSex<?Separator>
	| "\o"  DigitSequenceOct<?Separator>
	| "\d"? DigitSequenceDec<?Separator>
	| "\x"  DigitSequenceHex<?Separator>
	| "\z"  DigitSequenceNif<?Separator>
;

DigitSequenceBin<Separator> :::= ( DigitSequenceBin<?Separator> . <Separator+>"_"? )? [0-1];
DigitSequenceQua<Separator> :::= ( DigitSequenceQua<?Separator> . <Separator+>"_"? )? [0-3];
DigitSequenceSex<Separator> :::= ( DigitSequenceSex<?Separator> . <Separator+>"_"? )? [0-5];
DigitSequenceOct<Separator> :::= ( DigitSequenceOct<?Separator> . <Separator+>"_"? )? [0-7];
DigitSequenceDec<Separator> :::= ( DigitSequenceDec<?Separator> . <Separator+>"_"? )? [0-9];
DigitSequenceHex<Separator> :::= ( DigitSequenceHex<?Separator> . <Separator+>"_"? )? [0-9a-f];
DigitSequenceNif<Separator> :::= ( DigitSequenceNif<?Separator> . <Separator+>"_"? )? [0-9a-z];
```

### Syntax
```diff
+// NATURAL ::= [./lexicon.ebnf#Natural];
 // INTEGER ::= [./lexicon.ebnf#Integer];

PrimitiveLiteral ::=
+	| NATURAL
	| INTEGER
	| FLOAT
	| STRING
	| KeywordValue
	| "@" Word
;
```

## Operators
The unary prefix negation operator, `-`, is invalid for natural number types. Using it on them raises a type error.

The subtraction operator, `a - b`, when used for `nat` values, when the minuend (the first number `a`) is less than the subtrahend (the second number `b`), the result will always be `+0`. For example `+4 - +6` will evaluate to `+0`, even though `4 - 6` evaluates to `-2`. This is akin to truncating the decimal when dividing ints (e.g. `3 / 2 == 1` even though `3.0 / 2.0 == 1.5`).

Division of nats behaves the same as division for ints, truncating the decimal. `+5 / +3` is `+1`, not `+2` (even though 2 is mathematically closer to the actual value).

A new keyword unary prefix operator, `nat`, casts its numeric operand to the natural number type. It has the same precedence as the `int` and `float` keyword operators: weaker than symbolic operators, and right-to-left grouped (`nat int 4.2` is parsed as `nat (int 4.2)`). If the operand is not numeric, a type error is raised.

- When converting ints to nats, its 64 bits are simply interpreted as unsigned: any int value between -(2^63) and -1 is just added mathematically to 2^64. No change in binary representation is done.
- When converting nats to ints, the opposite occurs: if the nat is less than 2^63, an equivalent int is returned; otherwise 2^64 is subtracted mathematically from it but the bits are not changed.
- When converting floats to nats, the same algorithm as float-to-int is used (“round-towards-zero”), except that any negative-valued floats will convert to the nat +0. Floating-point values greater than the maximal nat will return the maximal nat. Unrepresentable values and NaN etc. will raise an error.
- When converting nats to floats, some precision will be lost for natural numbers greater than *2^53*. See the *IEEE 754* spec for details.

### Lexicon
```diff
KeywordOp :::=
+	| "nat"
	| "int"
	| "float"
;
```

### Semantics
```diff
SemanticOperation[operator: NEG]
-	::= SemanticExpression[type: Number];
+	::= SemanticExpression[type: Number - Natural];

-SemanticOperation[operator: EXP | MUL | DIV | ADD       | LT | LE | GT | GE | IS]
+SemanticOperation[operator: EXP | MUL | DIV | ADD | SUB | LT | LE | GT | GE | IS]
	::= SemanticExpression[type: Number] SemanticExpression[type: Number];
```

### Decorate
```diff
Decorate(ExpressionUnaryKeyword ::= ExpressionUnarySymbol) -> SemanticExpression
	:= Decorate(ExpressionUnarySymbol);
+Decorate(ExpressionUnaryKeyword ::= "nat" ExpressionUnaryKeyword) -> SemanticOperation
+	:= (SemanticOperation[operator=NAT]
+		Decorate(ExpressionUnaryKeyword)
+	);
Decorate(ExpressionUnaryKeyword ::= "int" ExpressionUnaryKeyword) -> SemanticOperation
	:= (SemanticOperation[operator=INT]
		Decorate(ExpressionUnaryKeyword)
	);
Decorate(ExpressionUnaryKeyword ::= "float" ExpressionUnaryKeyword) -> SemanticOperation
	:= (SemanticOperation[operator=FLOAT]
		Decorate(ExpressionUnaryKeyword)
	);

Decorate(ExpressionAdditive ::= ExpressionAdditive "-" ExpressionMultiplicative) -> SemanticOperation
-	:= (SemanticOperation[operator=ADD]
-		Decorate(ExpressionAdditive)
-		(SemanticOperation[operator=NEG] Decorate(ExpressionMultiplicative))
-	);
+	:= (SemanticOperation[operator=SUB]
+		Decorate(ExpressionAdditive)
+		Decorate(ExpressionMultiplicative)
+	);
```

#### TypeOf
```diff
Type! TypeOfUnfolded(SemanticOperation[operator: NEG] expr) :=
	1. *Assert:* `expr.children.count` is 1.
	2. *Let* `t0` be *Unwrap:* `TypeOf(expr.children.0)`.
	3. *If* *UnwrapAffirm:* `IsBottomType(t0)` is `true`:
		1. *Return:* `Never`.
	4. *If* *UnwrapAffirm:* `Subtype(t0, Number)` is `true`:
-		1. *Return:* `t0`.
+		1. *If* *UnwrapAffirm:* `Intersection(t0, Natural)` is `never`:
+			1. *Return:* `t0`.
	5. *Throw:* a new TypeErrorInvalidOperation.
;

-Type! TypeOfUnfolded(SemanticOperation[operator:       INT | FLOAT] expr) :=
+Type! TypeOfUnfolded(SemanticOperation[operator: NAT | INT | FLOAT] expr) :=
	1. *Assert:* `expr.children.count` is 1.
	2. *Let* `t0` be *Unwrap:* `TypeOf(expr.children.0)`.
	3. *If* *UnwrapAffirm:* `IsBottomType(t0)` is `true`:
		1. *Return:* `Never`.
	4. *If* *UnwrapAffirm:* `Subtype(t0, Number)` is `true`:
+		1. *If* `operator` is `NAT`:
+			1. *Return:* `Natural`.
-		2. *If* `operator` is `INT`:
+		2. *Else If* `operator` is `INT`:
			1. *Return:* `Integer`.
		3. *Else:*
			1. *Assert:* `operator` is `FLOAT`.
			2. *Return:* `Float`.
	5. *Throw:* a new TypeErrorInvalidOperation.
;

-Type! TypeOfUnfolded(SemanticOperation[operator: EXP | MUL | DIV | ADD]       expr) :=
+Type! TypeOfUnfolded(SemanticOperation[operator: EXP | MUL | DIV | ADD | SUB] expr) :=
	1. *Assert:* `expr.children.count` is 2.
	2. *Let* `t0` be *Unwrap:* `TypeOf(expr.children.0)`.
	3. *Let* `t1` be *Unwrap:* `TypeOf(expr.children.1)`.
+	4. *If* *UnwrapAffirm:* `Subtype(t0, Natural)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Natural)` is `true`:
+		1. *Return:* `Natural`.
-	4. *If* *UnwrapAffirm:* `Subtype(t0, Integer)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Integer)` is `true`:
+	4. *Else If* *UnwrapAffirm:* `Subtype(t0, Integer)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Integer)` is `true`:
		1. *Return:* `Integer`.
	5. *Else If* *UnwrapAffirm:* `Subtype(t0, Float)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Float)` is `true`:
		1. *Return:* `Float`.
	6. *Throw:* a new TypeErrorInvalidOperation.
;

Type! TypeOfUnfolded(SemanticOperation[operator: LT | GT | LE | GE] expr) :=
	1. *Assert:* `expr.children.count` is 2.
	2. *Let* `t0` be *Unwrap:* `TypeOf(expr.children.0)`.
	3. *Let* `t1` be *Unwrap:* `TypeOf(expr.children.1)`.
+	4. *If* *UnwrapAffirm:* `Subtype(t0, Natural)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Natural)` is `true`:
+		1. *Return:* `Boolean`.
-	4. *If* *UnwrapAffirm:* `Subtype(t0, Integer)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Integer)` is `true`:
+	4. *Else If* *UnwrapAffirm:* `Subtype(t0, Integer)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Integer)` is `true`:
		1. *Return:* `Boolean`.
	5. *Else If* *UnwrapAffirm:* `Subtype(t0, Float)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Float)` is `true`:
		1. *Return:* `Boolean`.
	6. *Throw:* a new TypeErrorInvalidOperation.
;
```

#### ValueOf
```diff
-Number! ValueOf(SemanticOperation[operator: EXP | MUL | DIV | ADD]       expr) :=
+Number! ValueOf(SemanticOperation[operator: EXP | MUL | DIV | ADD | SUB] expr) :=
	1. *Assert:* `expr.children.count` is 2.
	2. *Let* `v0` be *Unwrap:* `ValueOf(expr.children.0)`.
	3. *Let* `v1` be *Unwrap:* `ValueOf(expr.children.1)`.
	4. *Assert:* `v0` is an instance of `Number` *and* `v1` is an instance of `Number`.
	5. *Return:* `PerformBinaryArithmetic(expr.operator, v0, v1)`.
;

Number! PerformBinaryArithmetic(Text op, Number operand0, Number operand1) :=
	1. *If* `op` is `EXP`:
		1. *Let* `result` be the power, `operand0 ^ operand1`,
			obtained by raising `operand0` (the base) to `operand1` (the exponent).
		2. *Return:* `result`.
	2. *Else If* `op` is `MUL`:
		1. *Let* `result` be the product, `operand0 * operand1`,
			obtained by multiplying `operand0` (the multiplicand) by `operand1` (the multiplier).
		2. *Return:* `result`.
	3. *Else If* `op` is `DIV`:
		1. *Let* `result` be the quotient, `operand0 / operand1`,
			obtained by dividing `operand0` (the dividend) by `operand1` (the divisor).
		2. *Return:* `result`.
	4. *Else If* `op` is `ADD`:
		1. *Let* `result` be the sum, `operand0 + operand1`,
			obtained by adding `operand0` (the augend) to `operand1` (the addend).
		2. *Return:* `result`.
+	5. *Else If* `op` is `SUB`:
+		1. *Let* `result` be the difference, `operand0 - operand1`,
+			obtained by subtracting `operand1` (the subtrahend) from `operand0` (the minuend).
+		2. *Return:* `result`.
	6. *Throw:* a new TypeErrorInvalidOperation.
```

#### Build
```diff
-Sequence<Instruction> BuildExpression(SemanticOperation[operator: EXP | MUL | DIV | ADD       | LT | GT | LE | GE | ID | EQ] expr) :=
+Sequence<Instruction> BuildExpression(SemanticOperation[operator: EXP | MUL | DIV | ADD | SUB | LT | GT | LE | GE | ID | EQ] expr) :=
	1. *Let* `builds` be *UnwrapAffirm:* `PrebuildSemanticOperationBinary(expr)`.
	2. *Return:* [
		...builds.0,
		...builds.1,
		"`expr.operator`",
	].
;
```
