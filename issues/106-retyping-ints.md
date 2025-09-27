Revamping the Integer type and compatibility with Floats.

The first and biggest change is that the `int` type is now 64-bit (was previously 16-bit) two’s complement. Integers now range from *-(2^63)* to *2^63 - 1* (from *-9,223,372,036,854,775,808* to *9,223,372,036,854,775,807*).

The second big change is that ints and floats no longer work together implicitly. That is, ints are not automatically coerced into floats when operated on with them. Instead, explicit casting must be done.
```cp
claim y: int;
let x: float = 42;      % no longer allowed --- now a TypeError
let x: float = 42.0;    % good alternative for numeric literals
let x: float = float y; % general alternative: the new `float` cast operator (see below)
```

Arithmetic operations no longer implicitly coerce ints when mixed with floats. The following example operations are all **type errors**. Use explicit casting as above.
- `42 + 3.1`
- `42 * 3.1`
- `7.0 / 2`
- `5.0 < 4`

(For completeness), all of the following operators will now behave this way:
- exponentiation `^`
- multiplication `*`, division `/`
- addition `+`, subtraction `-`
- less than `<`, greater than `>`, less or equal `<=`, greater or equal `>=`, not less than `!<`, not greater than `!>`

Division `a / b` will *either*: only accept two floating-point operands, and it will never truncate the result; *or* only accept two integer operands, and will always truncate the result. Mixing types is not allowed.
- `7 / 2 == 3`, `-7 / 2 == -3` — When the fractional part is truncated, always rounds toward 0.
- `7.0 / 2.0 == 3.5`, `-7.0 / 2.0 == -3.5` — When both operands are floats, no truncation is made.
- If the divisor (the denominator, the second operand) is `0` (int) or `-0.0` or `0.0` (floats), then a compile-time or runtime error is thrown.

Equality `a == b` still coerces ints to floats when they are being compared, e.g. `42 == 42.0` is valid and will produce `true`, and `42 == 42.5` is also valid but will produce `false`. (Equivalent but opposite logic exists for `!=`.) Constant folding: `a == b` will compile to `false` when the typer can determine that `a` and `b`  will never be equal no matter what (e.g., their types are not numeric and they don’t overlap). Similarly, the expression will compile to `true` if the compiler can determine that `a` and `b` are always equal. Otherwise its type is `bool` and a VM instruction is compiled.

The remaining binary operators also still accept mixed types. This hasn’t changed.
- The object instanceof operators `is`, `isnt`
- the identity operators `===`, `!==`
- the logical operators `&&`, `!&`, `||`, `!|`

### TypeOf
```diff
Type! TypeOfUnfolded(SemanticOperation[operator: EXP | MUL | DIV | ADD] expr) :=
	1. *Assert:* `expr.children.count` is 2.
	2. *Let* `t0` be *Unwrap:* `TypeOf(expr.children.0)`.
	3. *Let* `t1` be *Unwrap:* `TypeOf(expr.children.1)`.
-	4. *If* *UnwrapAffirm:* `Subtype(t0, Number)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Number)` is `true`:
-		1. *If* *UnwrapAffirm:* `Subtype(t0, Float)` is `true` *or* *UnwrapAffirm:* `Subtype(t1, Float)` is `true`:
-			1. *Return:* `Float`.
-		2. *Else:*
-			1. *Return:* `Integer`.
+	4. *If* *UnwrapAffirm:* `Subtype(t0, Integer)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Integer)` is `true`:
+		1. *Return:* `Integer`.
+	5. *Else If* *UnwrapAffirm:* `Subtype(t0, Float)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Float)` is `true`:
+		1. *Return:* `Float`.
	6. *Throw:* a new TypeErrorInvalidOperation.
;

Type! TypeOfUnfolded(SemanticOperation[operator: LT | GT | LE | GE] expr) :=
	1. *Assert:* `expr.children.count` is 2.
	2. *Let* `t0` be *Unwrap:* `TypeOf(expr.children.0)`.
	3. *Let* `t1` be *Unwrap:* `TypeOf(expr.children.1)`.
-	4. *If* *UnwrapAffirm:* `Subtype(t0, Number)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Number)` is `true`:
-		1. *Return:* `Boolean`.
+	4. *If* *UnwrapAffirm:* `Subtype(t0, Integer)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Integer)` is `true`:
+		1. *Return:* `Boolean`.
+	5. *Else If* *UnwrapAffirm:* `Subtype(t0, Float)` is `true` *and* *UnwrapAffirm:* `Subtype(t1, Float)` is `true`:
+		1. *Return:* `Boolean`.
	6. *Throw:* a new TypeErrorInvalidOperation.
;
```

## New `int` and `float` Keyword Operators
The keywords `int` and `float`, when present in an expression (not a type), are now unary prefix operators. They convert their numeric operand into their respective type. If the operand is not numeric, a type error is raised.
```cp
let my_int: int   = 2;
let my_flt: float = -3.5;

2 * int my_flt;     % converts -3.5 to -3, then compiles to `2 * -3`
7.0 / float my_int; % converts 2 to 2.0, then compiles to `7.0 / 2.0`

int   my_int; % does nothing
float my_flt; % does nothing
```

- When converting floats to ints, the “round-toward-zero” (truncation) method is used. Both `-0.0` and `0.0` convert to `0`. If the float is greater than the maximal int, the maximal int is returned. (Similar for minimal int.) For NaN and other unrepresentable values, an error is raised.
- When converting ints to floats, some precision will be lost for integers greater than *2^53*. See the *IEEE 754* spec for details.

Per the language grammar, keyword unary prefix operators are *weaker* than symbolic ones. Therefore `int -my_flt` and `-(int my_flt)` are well-formed, whereas `-int my_flt` is not. All unary prefix operators are grouped right-to-left.

```cp
float int 2.25; %== float (int 2.25) %== 2.0

int null;            %> TypeError
float false;         %> TypeError
int "hello";         %> TypeError
float "world";       %> TypeError
int [0.2];           %> TypeError
float [myRecord= 2]; %> TypeError

claim my_number: int | float;
int   my_number; % ok
float my_number; % ok

claim maybe_float: float | null;
int maybe_float; %> TypeError
```

### Lexicon
```diff
+KeywordOp :::=
+	| "int"
+	| "float"
+;
```

### Syntax
```diff
+// KEYWORD_OP ::= [./lexicon.ebnf#KeywordOp];

ExpressionUnarySymbol
	::= ExpressionCompound | ("!" | "?" | "+" | "-") ExpressionUnarySymbol;

+ExpressionUnaryKeyword
+	::= ExpressionUnarySymbol | KEYWORD_OP ExpressionUnaryKeyword;

ExpressionCast ::=
-	| (ExpressionCast ("as" | "as?" | "as!"))? ExpressionUnarySymbol
+	| (ExpressionCast ("as" | "as?" | "as!"))? ExpressionUnaryKeyword
	|  ExpressionCast  "as"                    "<" Type ">"
;
```

### Decorate
```diff
+Decorate(ExpressionUnaryKeyword ::= ExpressionUnarySymbol) -> SemanticExpression
+	:= Decorate(ExpressionUnarySymbol);
+Decorate(ExpressionUnaryKeyword ::= "int" ExpressionUnaryKeyword) -> SemanticOperation
+	:= (SemanticOperation[operator=INT]
+		Decorate(ExpressionUnaryKeyword)
+	);
+Decorate(ExpressionUnaryKeyword ::= "float" ExpressionUnaryKeyword) -> SemanticOperation
+	:= (SemanticOperation[operator=FLOAT]
+		Decorate(ExpressionUnaryKeyword)
+	);
```

#### TypeOf
```diff
+Type! TypeOfUnfolded(SemanticOperation[operator: INT | FLOAT] expr) :=
+	1. *Assert:* `expr.children.count` is 1.
+	2. *Let* `t0` be *Unwrap:* `TypeOf(expr.children.0)`.
+	3. *If* *UnwrapAffirm:* `IsBottomType(t0)` is `true`:
+		1. *Return:* `Never`.
+	4. *If* *UnwrapAffirm:* `Subtype(t0, Number)` is `true`:
+		1. *If* `operator` is `INT`:
+			1. *Return:* `Integer`.
+		2. *Else:*
+			1. *Assert:* `operator` is `FLOAT`.
+			2. *Return:* `Float`.
+	5. *Throw:* a new TypeErrorInvalidOperation.
+;
```

#### ValueOf
```diff
Number! ValueOf(SemanticOperation[operator: EXP | MUL | DIV | ADD] expr) :=
	1. *Assert:* `expr.children.count` is 2.
	2. *Let* `v0` be *Unwrap:* `ValueOf(expr.children.0)`.
	3. *Let* `v1` be *Unwrap:* `ValueOf(expr.children.1)`.
	4. *Assert:* `v0` is an instance of `Number` *and* `v1` is an instance of `Number`.
-	5. *If* `v0` is an instance of `Integer` *and* `v1` is an instance of `Integer`:
-		1. *Let* `number` be *UnwrapAffirm:* `PerformBinaryArithmetic(expr.operator, v0, v1)`.
-		2. *Return:* `Integer(number)`.
-	6. *Else:*
-		1. *Let* `number` be *UnwrapAffirm:* `PerformBinaryArithmetic(expr.operator, Float(v0), Float(v1))`.
-		2. *Return:* `Float(number)`.
+	5. *Return:* `PerformBinaryArithmetic(expr.operator, v0, v1)`.
;
```
