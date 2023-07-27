In record literals, allow key punning with augment and update operators. In mapping literals, allow antecedent punning. Dependent on #15 and #21.

# Descripiton

## Record Augment Assignment
In record literals, augmented assignment operators may be used as the delimiter of a key-value pair.
```
let a: int = 42;
let rec: [a: int] = [
	a /= 3,
];
% equivalent to:
% let rec: [a: int] = [
% 	a= a / 3,
% ];
```
This sets `rec.a` to `42 / 3`, but *does not modify* the unbound variable `a`, which still points to `42`. Whitespace may be inserted before the operator symbol to improve readability.

In record literals, update operators may be used.
```
let b: int = 42;
let rec: [b: int] = [
	b++,
];
% equivalent to:
% let rec: [b: int] = [
% 	b += 1,
% ];
```
This sets `rec.b` to `42 + 1`, but *does not modify* the unbound variable `b`, which still points to `42`.

## Record Punning
In a record literal, using the symbol `$` as a property value is equivalent to repeating the key. This is only valid when there exists an unbound variable or parameter whose identifier is exactly the same as the property key.
```
let c: int = 42;
let rec: [c: int] = [
	c= $,
];
% equivalent to:
% let rec: [c: int] = [
% 	c= c,
% ];
```
This sets `rec.c` to `42`.

Augment operators may also be used with punning:
```
let d: int = 42;
let rec: [d: int] = [
	d *= $,
];
% equivalent to:
% let rec: [d: int] = [
% 	d *= d,
% ];
```
This sets `rec.d` to `42 * 42`, but *does not modify* the unbound variable `d`, which still points to `42`.

The complete list of operators:
```
[a  ^= b] % sugar for `[a=  a  ^ b]`
[a  *= b] % sugar for `[a=  a  * b]`
[a  /= b] % sugar for `[a=  a  / b]`
[a  += b] % sugar for `[a=  a  + b]`
[a  -= b] % sugar for `[a=  a  - b]`
[a &&= b] % sugar for `[a=  a && b]`
[a !&= b] % sugar for `[a=  a !& b]`
[a ||= b] % sugar for `[a=  a || b]`
[a !|= b] % sugar for `[a=  a !| b]`

[a++] % sugar for `[a += 1]`
[a--] % sugar for `[a -= 1]`
[a**] % sugar for `[a *= 2]`
[a//] % sugar for `[a /= 2]`

[a = $] % sugar for `[a= a]`

[a  ^= $] % sugar for `[a  ^= a]`
[a  *= $] % sugar for `[a  *= a]`
[a  /= $] % sugar for `[a  /= a]`
[a  += $] % sugar for `[a  += a]`
[a  -= $] % sugar for `[a  -= a]`
[a &&= $] % sugar for `[a &&= a]`
[a !&= $] % sugar for `[a !&= a]`
[a ||= $] % sugar for `[a ||= a]`
[a !|= $] % sugar for `[a !|= a]`
```

## Mapping Punning
In a mapping literal, using the symbol `$` as the consequent of a case is equivalent to repeating the antecedent.
```
[
	42 |-> $,
];
% equivalent to:
% [
% 	42 |-> 42,
% ];
```

If a case has multiple antecedents, it deconstructs each of them.
```
[
	42, 43.0, '44' |-> $,
];
% equivalent to:
% [
% 	42   |-> 42,
% 	43.0 |-> 43.0,
% 	'44' |-> '44',
% ];
```

The complete list of operators:
```
[‹x›      |-> $]; % sugar for `[‹x› |-> ‹x›]`
[‹x›, ‹y› |-> $]; % sugar for `[‹x› |-> ‹x›, ‹y› |-> ‹y›]`
```
(where `‹x›` and `‹y›` are metavariables representing expressions)

# Specification

## Syntactic Grammar
```diff
AugmentOperator ::= "^="  | "*="  | "/=" | "+=" | "-=" | "&&=" | "||=";
AugmentNegate   ::= "!&=" | "!|=";
UpdateOperator  ::= "++"  | "--"  | "**" | "//";

Property ::=
	| Word       "=" Expression
+	| IDENTIFIER "=" "$"
+	| IDENTIFIER ((AugmentOperator | AugmentNegate) ("$" | Expression) | UpdateOperator)
;

-Case ::= Expression# "|->"        Expression;
+Case ::= Expression# "|->" ("$" | Expression);
```

## Decorate
```diff
Decorate(Property ::= Word "=" Expression) -> SemanticProperty
	:= (SemanticProperty
		Decorate(Word)
		Decorate(Expression)
	);
+Decorate(Property ::= IDENTIFIER "=" "$") -> SemanticProperty
+	:= (SemanticProperty
+		(SemanticKey[id=TokenWorth(IDENTIFIER)])
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+	);
+Decorate(Property ::= IDENTIFIER AugmentOperator "$") -> SemanticProperty
+	:= (SemanticProperty
+		(SemanticKey[id=TokenWorth(IDENTIFIER)])
+		(SemanticOperation[operator=Decorate(AugmentOperator)]
+			(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+			(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		)
+	);
+Decorate(Property ::= IDENTIFIER AugmentNegate "$") -> SemanticProperty
+	:= (SemanticProperty
+		(SemanticKey[id=TokenWorth(IDENTIFIER)])
+		(SemanticOperation[operator=NOT]
+			(SemanticOperation[operator=Decorate(AugmentNegate)]
+				(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+				(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+			)
+		)
+	);
+Decorate(Property ::= IDENTIFIER AugmentOperator Expression) -> SemanticProperty
+	:= (SemanticProperty
+		(SemanticKey[id=TokenWorth(IDENTIFIER)])
+		(SemanticOperation[operator=Decorate(AugmentOperator)]
+			(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+			Decorate(Expression)
+		)
+	);
+Decorate(Property ::= IDENTIFIER AugmentNegate Expression) -> SemanticProperty
+	:= (SemanticProperty
+		(SemanticKey[id=TokenWorth(IDENTIFIER)])
+		(SemanticOperation[operator=NOT]
+			(SemanticOperation[operator=Decorate(AugmentNegate)]
+				(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+				Decorate(Expression)
+			)
+		)
+	);
+Decorate(Property ::= IDENTIFIER UpdateOperator) -> SemanticProperty
+	:= (SemanticProperty
+		(SemanticKey[id=TokenWorth(IDENTIFIER)])
+		(SemanticOperation[operator=Decorate(UpdateOperator)]
+			(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+			(SemanticConstant[value=DefaultOperand(UpdateOperator)])
+		)
+	);

+Decorate(Case ::= Case__0__List "|->" "$") -> Sequence<SemanticCase>
+	:= ToCases(Case__0__List);
-Decorate(Case ::= Case__0__List "|->" Expression) -> SemanticCase
-	:= (SemanticCase
-		...Decorate(Case__0__List)
-		Decorate(Expression)
-	);
+Decorate(Case ::= Case__0__List "|->" Expression) -> Sequence<SemanticCase>
+	:= [
+		(SemanticCase
+			...Decorate(Case__0__List)
+			Decorate(Expression)
+		),
+	];

	Decorate(Case__0__List ::= Expression) -> Sequence<SemanticExpression>
		:= [Decorate(Expression)];
	Decorate(Case__0__List ::= Case__0__List "," Expression) -> Sequence<SemanticExpression>
		:= [
			...Decorate(Case__0__List),
			Decorate(Expression),
		];

+	ToCases(Case__0__List ::= Expression) -> Sequence<SemanticCase>
+		:= [
+			(SemanticCase
+				Decorate(Expression)
+				Decorate(Expression)
+			),
+		];
+	ToCases(Case__0__List ::= Case__0__List "," Expression) -> Sequence<SemanticCase>
+		:= [
+			...ToCases(Case__0__List),
+			(SemanticCase
+				Decorate(Expression)
+				Decorate(Expression)
+			),
+		];

Decorate(MappingLiteral__0__List ::= Case) -> Sequence<SemanticCase>
-	:= [Decorate(Case)];
+	:= Decorate(Case);
Decorate(MappingLiteral__0__List ::= MappingLiteral__0__List "," Case) -> Sequence<SemanticCase>
	:= [
		...Decorate(MappingLiteral__0__List),
-		Decorate(Case),
+		...Decorate(Case),
	];
```
