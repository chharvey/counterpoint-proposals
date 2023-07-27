Constant collections are a literal syntax of collections that contain only constants or other constant collections.

# Description
Constant collections are constructed via a specific syntax, but they also have certain semantics, as described in another issue. Constant collections may only take the form of Strings, Tuples, and Records; other data structures such as Lists, Hashes, Sets, and Maps may not be constant collections.

## Foldable Expressions
**Foldable expressions** are expressions whose values are either constant or computable by the compiler and thus are reduced to a single constant value at compile time. Foldable expressions are not enforced via a syntax production, as there is no way to determine syntactically whether the compiler can compute the value.

For example, if `x` is known by the compiler to be fixed and immutable at the value `[3]`, then the expression `x.0 + 2` is foldable, and the compiler will reduce the output to `5`. Since `x` is guaranteed to never be reassigned or modified, the compiler is able to make this optimization. This is called **Constant Folding** (#34). If `x` were unfixed or mutable, however, then the expression `x.0 + 2` would not be foldable, even though it’s syntactically identical. Any expression that contains primitive literals, fixed immutable variables, and operators is foldable, and any compound types made of foldable expressions is also foldable. E.g., using `x` above, this includes the Set `{2, x.0, x.0 + 2, x}`. String templates that interpolate only foldable expressions (or have no interpolation at all) are also foldable. *Some* function calls may be foldable, but only if the function can be executed at compile-time.

Expressions that contain unfixed or mutable variables, most function/method/constructor calls, function expressions (lambdas), or class expressions are *not* foldable.

## Constant Expressions
**Constant expressions**, by contrast, are a pure syntactic production. Constant expressions include only primitive literals, string literals, operators, template literals containing only constant expressions, and constant collections (below), which are special-syntaxed tuples and records that contain only constant expressions. Think of them as a more restrictive form of foldable expressions — All constant expressions are foldable, but not all foldable expressions are constant.

Constant expressions *do not* include variables of any kind (even if they are unfixed), regular tuples/records, other compound data types, function calls, or function/class expressions. Expressions that are not constant are called **variable**.

## Constant Collections
Constant collections are constant expressions that take the form of tuple/record literals whose entries are other constant expressions. They are written with the token `\` preceding the usual `[ … ]`. Nesting is allowed, but constant collections must not contain variable collections.

**Vects** are the tuple form of constant expressions. (“Vect” is short for “vector”, but not to be confused with the Vector Counterpoint Specification Type.)
```cp
\[1, 2.0, 'three', \[4]];
\[null, true, false, 'a string', '''a string template''', '''a {{ \['foldable'].[1 - 1] }} string template'''];
\[1 + 1, 2 * 2, 3.0 / 3.0, -4.0 ^ 4.0, true && false, null || 0];

let x: int = 42;
\[1, 2.0, 'three', [4]]; %> ParseError
\[40, 41, x];            %> ParseError
```
`[4]` and `x` are variable expressions and are not allowed inside a constant collection. This rule is enforced via syntactic analysis.

**Structs** are the record form of constant expressions. (“Struct” is short for “structure”, but again, not to be confused with the Structure Counterpoint Specification Type.)
```cp
\[
	first=  'who',
	second= \['what'],
	third=  \[i= 'don’t know'],
];
```

If a variable collection meets the criteria of a constant collection, that is, it contains only constant expressions, then converting the variable collection to a constant collection (prepending a `\` token) will reduce compilation time and produce a value object. For example, it is recommended to convert `['who', 'what', 'i don’t know']` into `\['who', 'what', 'i don’t know']` where possible.

The constant collection syntax parses and evaluates much faster in comparison to variable collections where there could be functions to call, members and identifiers to access, and pointers such as `this` and `super` to dereference.

## Semantics of Vects and Structs
Vects and Structs generate **value objects**. Value objects have no identity and are described only by their value. In contrast, **reference objects** are identified by reference.

As value objects, Vects and Structs are **immutable**: It’s impossible to change their entries, and it’s impossible to assign them to mutable types. Vects and Structs are very much like strings: even though a string is an array of characters, we cannot mutate it.
```cp
set \[42, 420, 4200].3 = 42000; %> MutabilityError

let values: mutable int[3] = \[42, 420, 4200];  %> TypeError03
let times10: mutable [n42: int, n420: int] = \[ %> TypeError03
	n42=  42,
	n420= 420,
];
```

However, it is possible to assign a Vect to a variable tuple type, as long as that type is not mutable. Same with Structs.
```cp
let values_tup: int[3] = \[42, 420, 4200];
values_tup.0; %== 42

let values_rec: [n42: int, n420: int] = \[
	n42=  42,
	n420= 420,
];
values_rec.n42; %== 42
```

As with all value objects, equality implies identity. Vects/Structs that have the same entries are considered identical: Comparing them with the `===` operator will return `true`. Vects/Structs that are equal are the same value.
```cp
let a: [int] = [42];
let b: [int] = [42];
a ==  b; % true      % They are “equal” because they contain equal items,
a === b; % false     % but they are not identical, since they aren’t the same object by reference.

let c: [int] = \[42];
let d: [int] = \[42];
c ==  d; % true       % They are “equal” because they contain equal items,
c === d; % true       % and they are also identical, since they are the same value.
```

# Specification

## Lexical Grammar
```diff
Punctuator ::=
	// grouping
-		| "[" | "]"
+		| "[" | "]" | "\["
;
```

## Syntactic Grammar
```diff
-StringTemplate ::=
+StringTemplate<Variable> ::=
	| TEMPLATE_FULL
-	| TEMPLATE_HEAD Expression?            (TEMPLATE_MIDDLE Expression?)*            TEMPLATE_TAIL
+	| TEMPLATE_HEAD Expression?<?Variable> (TEMPLATE_MIDDLE Expression<?Variable>?)* TEMPLATE_TAIL
;

-ExpressionGrouped            ::=                            "("        Expression                      ")";
-TupleLiteral                 ::=                            "[" ( ","? Expression#             ","? )? "]";
-RecordLiteral                ::=                            "["   ","? Property#               ","?    "]";
-SetLiteral                   ::=                            "{" ( ","? Expression#             ","? )? "}";
-MapLiteral                   ::=                            "{"   ","? Case#                   ","?    "}";
-FunctionArguments            ::=                            "(" ( ","? Expression#             ","? )? ")";
+ExpressionGrouped <Variable> ::=                            "("        Expression <?Variable>          ")";
+TupleLiteral      <Variable> ::= <Variable->"\[" <Variable+>"[" ( ","? Expression <?Variable># ","? )? "]";
+RecordLiteral     <Variable> ::= <Variable->"\[" <Variable+>"["   ","? Property   <?Variable># ","?    "]";
+SetLiteral                   ::=                            "{" ( ","? Expression <+Variable># ","? )? "}";
+MapLiteral                   ::=                            "{"   ","? Case       <+Variable># ","?    "}";
+FunctionArguments            ::=                            "(" ( ","? Expression <+Variable># ","? )? ")";

-ExpressionUnit ::=
+ExpressionUnit<Variable> ::=
-	| IDENTIFIER
+	| <Variable+>IDENTIFIER
	| PrimitiveLiteral
-	| StringTemplate
-	| ExpressionGrouped
-	| TupleLiteral
-	| RecordLiteral
-	| SetLiteral
-	| MapLiteral
+	| StringTemplate<?Variable>
+	| ExpressionGrouped<?Variable>
+	| TupleLiteral<-Variable>
+	| RecordLiteral<-Variable>
+	| <Variable+>TupleLiteral<?Variable>
+	| <Variable+>RecordLiteral<?Variable>
+	| <Variable+>SetLiteral
+	| <Variable+>MapLiteral
;

-PropertyAccess           ::= ("." | "?." | "!.") (INTEGER | Word | "[" Expression "]");
-PropertyAssign           ::=  "."                (INTEGER | Word | "[" Expression "]");
+PropertyAccess<Variable> ::= ("." | "?." | "!.") (INTEGER | Word | "[" Expression<?Variable> "]");
+PropertyAssign           ::=  "."                (INTEGER | Word | "[" Expression<+Variable> "]");
FunctionCall              ::=  "."                GenericArguments? FunctionArguments;

-ExpressionCompound ::=
-	| ExpressionUnit
-	| ExpressionCompound (PropertyAccess | FunctionCall)
+ExpressionCompound<Variable> ::=
+	| ExpressionUnit<?Variable>
+	| ExpressionCompound<?Variable> PropertyAccess<?Variable>
+	| <Variable+>(ExpressionCompound<?Variable> FunctionCall)
;

Assignee ::=
	| IDENTIFIER
-	| ExpressionCompound            PropertyAssign
+	| ExpressionCompound<+Variable> PropertyAssign
;
```

## Semantics
```diff
-SemanticTuple                   ::= SemanticExpression*;
-SemanticRecord                  ::= SemanticProperty+;
+SemanticTuple  [isRef: Boolean] ::= SemanticExpression*;
+SemanticRecord [isRef: Boolean] ::= SemanticProperty+;
```

## Decorate
```diff
-Decorate(ExpressionGrouped ::= "(" Expression ")") -> SemanticExpression
-	:= Decorate(Expression);
+Decorate(ExpressionGrouped<Variable> ::= "(" Expression<?Variable> ")") -> SemanticExpression
+	:= Decorate(Expression<?Variable>);

-Decorate(TupleLiteral ::= "[" "]") -> SemanticTuple
-	:= (SemanticTuple);
+Decorate(TupleLiteral<-Variable> ::= "\[" "]") -> SemanticTuple
+	:= (SemanticTuple[isRef=false]);
+Decorate(TupleLiteral<+Variable> ::= "[" "]") -> SemanticTuple
+	:= (SemanticTuple[isRef=true]);

-Decorate(TupleLiteral ::= "[" ","? Expression# ","? "]") -> SemanticTuple
-	:= (SemanticTuple
-		...ParseList(Expression, SemanticExpression)
-	);
+Decorate(TupleLiteral<-Variable> ::= "\[" ","? Expression<?Variable># ","? "]") -> SemanticTuple
+	:= (SemanticTuple[isRef=false]
+		...ParseList(Expression<?Variable>, SemanticExpression)
+	);
+Decorate(TupleLiteral<+Variable> ::= "[" ","? Expression<?Variable># ","? "]") -> SemanticTuple
+	:= (SemanticTuple[isRef=true]
+		...ParseList(Expression<?Variable>, SemanticExpression)
+	);

-Decorate(RecordLiteral ::= "[" ","? Property# ","? "]") -> SemanticRecord
-	:= (SemanticRecord
-		...ParseList(Property, SemanticProperty)
-	);
+Decorate(RecordLiteral<-Variable> ::= "\[" ","? Property<?Variable># ","? "]") -> SemanticRecord
+	:= (SemanticRecord[isRef=false]
+		...ParseList(Property<?Variable>, SemanticProperty)
+	);
+Decorate(RecordLiteral<+Variable> ::= "[" ","? Property<?Variable># ","? "]") -> SemanticRecord
+	:= (SemanticRecord[isRef=true]
+		...ParseList(Property<?Variable>, SemanticProperty)
+	);

-Decorate(ExpressionUnit ::= IDENTIFIER) -> SemanticVariable
-	:= (SemanticVariable[id=TokenWorth(IDENTIFIER)]);
-Decorate(ExpressionUnit ::= PrimitiveLiteral) -> SemanticConstant
-	:= Decorate(PrimitiveLiteral);
-Decorate(ExpressionUnit ::= StringTemplate) -> SemanticTemplate
-	:= Decorate(StringTemplate);
-Decorate(ExpressionUnit ::= ExpressionGrouped) -> SemanticExpression
-	:= Decorate(ExpressionGrouped);
-Decorate(ExpressionUnit ::= TupleLiteral) -> SemanticTuple
-	:= Decorate(TupleLiteral);
-Decorate(ExpressionUnit ::= RecordLiteral) -> SemanticRecord
-	:= Decorate(RecordLiteral);
-Decorate(ExpressionUnit ::= SetLiteral) -> SemanticSet
-	:= Decorate(SetLiteral);
-Decorate(ExpressionUnit ::= MapLiteral) -> SemanticMap
-	:= Decorate(MapLiteral);
+Decorate(ExpressionUnit<Variable> ::= <Variable+>IDENTIFIER) -> SemanticVariable
+	:= (SemanticVariable[id=TokenWorth(IDENTIFIER)]);
+Decorate(ExpressionUnit<Variable> ::= PrimitiveLiteral) -> SemanticConstant
+	:= Decorate(PrimitiveLiteral);
+Decorate(ExpressionUnit<Variable> ::= StringTemplate<?Variable>) -> SemanticTemplate
+	:= Decorate(StringTemplate<?Variable>);
+Decorate(ExpressionUnit<Variable> ::= ExpressionGrouped<?Variable>) -> SemanticExpression
+	:= Decorate(ExpressionGrouped<?Variable>);
+Decorate(ExpressionUnit<Variable> ::= TupleLiteral<-Variable>) -> SemanticTuple
+	:= Decorate(TupleLiteral<-Variable>);
+Decorate(ExpressionUnit<Variable> ::= RecordLiteral<-Variable>) -> SemanticRecord
+	:= Decorate(RecordLiteral<-Variable>);
+Decorate(ExpressionUnit<Variable> ::= <Variable+>TupleLiteral<?Variable>) -> SemanticTuple
+	:= Decorate(TupleLiteral<?Variable>);
+Decorate(ExpressionUnit<Variable> ::= <Variable+>RecordLiteral<?Variable>) -> SemanticRecord
+	:= Decorate(RecordLiteral<?Variable>);
+Decorate(ExpressionUnit<+Variable> ::= SetLiteral) -> SemanticSet
+	:= Decorate(SetLiteral);
+Decorate(ExpressionUnit<+Variable> ::= MapLiteral) -> SemanticMap
+	:= Decorate(MapLiteral);
```

## TypeOf
```diff
Type! TypeOfUnfolded(SemanticTuple tuple) :=
	1. *Let* `types` be a mapping of `tuple` for each `it` to *Unwrap:* `TypeOf(it)`.
-	2. *Return:* a new mutable Tuple type containing the items in the sequence `types`.
+	2. *If* `tuple.isRef` is `false`:
+		1. *Return:* a new non-mutable Tuple type containing the items in the sequence `types`.
+	3. *Else:*
+		1. *Return:* a new mutable Tuple type containing the items in the sequence `types`.
;

Type! TypeOfUnfolded(SemanticRecord record) :=
	1. *Let* `types` be a new Structure.
	2. *For each* `property` in `record`:
		1. *Set* `types` to a new Structure [
			...types,
			`property.children.0` = *Unwrap:* `TypeOf(property.children.1)`,
		].
-	3. *Return:* a new mutable Record type containing the properties in the structure `types`.
+	3. *If* `record.isRef` is `false`:
+		1. *Return:* a new non-mutable Record type containing the properties in the structure `types`.
+	4. *Else:*
+		1. *Return:* a new mutable Record type containing the properties in the structure `types`.
;
```

## Intrinsics
```diff
+## `Vect`
+`Vect` objects have the same shape as `Tuple` objects, except `Vect` objects are value objects.

+## `Struct`
+`Struct` objects have the same shape as `Record` objects, except `Struct` objects are value objects.
```

## ValueOf
```diff
-Tuple! ValueOf(SemanticTuple tuple) :=
+Or<Vect, Tuple>! ValueOf(SemanticTuple tuple) :=
	1. *Let* `data` be a mapping of `tuple` for each `it` to *Unwrap:* `ValueOf(it)`.
-	2. *Return:* a new Tuple containing the items in the sequence `data`.
+	2. *If* `tuple.isRef` is `false`:
+		1. *Return:* a new Vect containing the items in the sequence `data`.
+	3. *Else:*
+		1. *Return:* a new Tuple containing the items in the sequence `data`.
;

-Record! ValueOf(SemanticRecord record) :=
+Or<Struct, Record>! ValueOf(SemanticRecord record) :=
	1. *Let* `data` be a new Structure.
	2. *For each* `property` in `record`:
		1. Set `data` to a new Structure [
			...data,
			`property.children.0.id`= *Unwrap:* `ValueOf(property.children.1)`,
		].
-	3. *Return:* a new Record containing the properties in the structure `data`.
+	3. *If* `record.isRef` is `false`:
+		1. *Return:* a new Struct containing the properties in the structure `data`.
+	4. *Else:*
+		1. *Return:* a new Record containing the properties in the structure `data`.
;
```

## Identical
```diff
Boolean Identical(Object a, Object b) :=
#	...
	6. *If* `a` is an instance of `String` *and* `b` is an instance of `String`:
		1. *If* `a` and `b` are exactly the same sequence of code units
			(same length and same code units at corresponding indices):
			1. *Return:* `true`.
+	7. *If* `a` is an instance of `Vect` *and* `b` is an instance of `Vect`:
+		1. *Perform:* Step 4 listed in `Equal(a, b)`.
+	8. *If* `a` is an instance of `Struct` *and* `b` is an instance of `Struct`:
+		1. *Perform:* Step 5 listed in `Equal(a, b)`.
	9. *If* `a` and `b` are the same object:
		1. *Return:* `true`.
	10. Return `false`.
;
```

## Equal
```diff
Boolean Equal(Object a, Object b) :=
#	...
-	4. *If* `a` is an instance of `Tuple`           or `List` *and* `b` is an instance of `Tuple`           or `List`:
+	4. *If* `a` is an instance of `Tuple` or `Vect` or `List` *and* `b` is an instance of `Tuple` or `Vect` or `List`:
#		...
-	5. *If* `a` is an instance of `Record`             or `Dict` *and* `b` is an instance of `Record`             or `Dict`:
+	5. *If* `a` is an instance of `Record` or `Struct` or `Dict` *and* `b` is an instance of `Record` or `Struct` or `Dict`:
#		...
#	...
;
```
