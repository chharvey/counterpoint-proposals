Dot-Access entries of tuples and records:

- Access a tuple item with an index literal: `tuple.1`.
- Access a record value with a word: `record.key`.

Computed-Access entries of tuples, records, and mappings:

- Access a tuple item with a computed integer: `tuple.[3 - 2]`.
- *DRAFT: Access a record value with an atom: `record.[if first then .key1 else .key2]`.*
- Access a mapping consequent with a computed expression: `mapping.[5 + 3]`, `mapping.['value']`, `mapping.[[x = 5, y = 3]]`, etc.

*DRAFT: Using atoms (#59) as record keys:*

Atoms can be used as computed keys of records. The key name is the atom name.
```cp
type Font = [
	weight: int,
	size:   float,
	style:  .BOLD | .ITALIC | .OBLIQUE,
];
let font: Font = [
	weight= 400,
	size=   12.0,
	style=  .ITALIC,
];
font.[.weight] == 400;
let sz: .size = .size;
font.[sz] == 12.0;
font.[if font.size < 24 then .style else .sTyLe] == .ITALIC;
```

Nonexistent keys will result in an VoidError.
```cp
font.sTyLe;    %> TypeError04
font.[.sTyLe]; % no compiler error, but runtime VoidError
```
This is similar to the behavior of tuples if we try to access a nonexistent index:
```cp
let tuple: [int, float] = [400, 18.5];
tuple.2;   %> TypeError04
tuple.[2]; %> no compiler error, but runtime VoidError
```

# Specification
## TypeError04
A `TypeError04` is thrown when the validator encounters a non-existent index or property access. The standard code for this error is *2304* and the standard message is *'Index|Property `<iden>` does not exist on type `<type>`.'*.

## VoidError
`VoidError` is thrown when the runtime virtual machine attempts to access a value that does not exist. This happens for example when an entry of a collection is accessed where there is no value. The standard code for this error is *3100* and the standard message is *'Value is undefined.'*.

## Lexical Grammar
```diff
Punctuator :::=
	// expression
+		| "."
```

## Syntactic Grammar
```diff
+PropertyAccess
+	::= "." (INTEGER | Word | "[" Expression "]");

+ExpressionCompound ::=
+	| ExpressionUnit
+	| ExpressionCompound PropertyAccess
+;

ExpressionUnarySymbol ::=
-	| ExpressionUnit
+	| ExpressionCompound
	| ("!" | "?" | "+" | "-") ExpressionUnarySymbol;
```

## Semantic Schema
```diff
SemanticExpression =:=
	| SemanticConstant
	| SemanticVariable
	| SemanticTemplate
+	| SemanticAccess
	| SemanticOperation
;

SemanticKey[id: RealNumber]
	::= ();

+SemanticIndex
+	::= SemanticConstant;

+SemanticAccess
+	::= SemanticExpression (SemanticIndex | SemanticKey | SemanticExpression);
```

## Decorate
```diff
+Decorate(PropertyAccess ::= "." INTEGER) -> SemanticIndex
+	:= (SemanticIndex
+		(SemanticConstant[value=Integer(TokenWorth(INTEGER))])
+	);
+Decorate(PropertyAccess ::= "." Word) -> SemanticKey
+	:= Decorate(Word);
+Decorate(PropertyAccess ::= "." "[" Expression "]") -> SemanticExpression
+	:= Decorate(Expression);

+Decorate(ExpressionCompound ::= ExpressionUnit) -> SemanticExpression
+	:= Decorate(ExpressionUnit);
+Decorate(ExpressionCompound ::= ExpressionCompound PropertyAccess) -> SemanticAccess
+	:= (SemanticAccess
+		Decorate(ExpressionCompound)
+		Decorate(PropertyAccess)
+	);

-Decorate(ExpressionUnarySymbol ::= ExpressionUnit) -> SemanticExpression
-	:= Decorate(ExpressionUnit);
+Decorate(ExpressionUnarySymbol ::= ExpressionCompound) -> SemanticExpression
+	:= Decorate(ExpressionCompound);
```

## TypeOf
```
Type! TypeOfUnassessed(SemanticAccess access) :=
	1. *Assert:* `access.children.count` is 2.
	2. *Let* `base` be `access.children.0`.
	3. *Let* `accessor` be `access.children.1`.
	4. *Let* `base_type` be *Unwrap:* `TypeOf(base)`.
	5. *If* `accessor` is a SemanticIndex:
		1. *Assert:* `accessor.children.count` is 1.
		2. *Let* `accessor_type` be *Unwrap:* `TypeOf(accessor.children.0)`.
		3. *Assert:* `accessor_type` is a TypeConstant.
		4. *Let* `i` be `accessor_type.value`.
		5. *Assert:* `i` is a instance of `Integer`.
		6. *If* *UnwrapAffirm:* `Subtype(base_type, Tuple)` *and* `i` is an index in `base_type`:
			1. *Return:* the type accessed at index `i` in `base_type`.
		7. *Throw:* a new TypeError04 "Index {{ i }} does not exist on type {{ base_type }}.".
	6. *If* `accessor` is a SemanticKey:
		1. *Let* `id` be `accessor.id`.
		2. *If* *UnwrapAffirm:* `Subtype(base_type, Record)` *and* `id` is a key in `base_type`:
			1. *Return:* the type accessed at key `id` in `base_type`.
		3. *Throw:* a new TypeError04 "Property {{ id }} does not exist on type {{ base_type }}.".
	7. *Assert:* `accessor` is a SemanticExpression.
	8. *Let* `accessor_type` be *Unwrap:* `TypeOf(accessor)`.
	9. *If* *UnwrapAffirm:* `Subtype(base_type, Tuple)`:
		1. *If* `accessor_type` is a TypeConstant:
			1. *Let* `i` be `accessor_type.value`.
			2. *If* `i` is a instance of `Integer` *and* `i` is an index in `base_type`:
				1. *Return:* the type accessed at index `i` in `base_type`.
		2. *Else If* *UnwrapAffirm:* `Subtype(accessor_type, Integer)`:
			1. *Return:* the union of all item types in `base_type`.
		3. *Throw:* a new TypeError02 "Type {{ accessor_type }} is not a subtype of type {{ Integer }}.".
	10. *If* *UnwrapAffirm:* `Subtype(base_type, Mapping)`:
		1. *Let* `k` be the type of the antecedents in `base_type`.
		2. *Let* `v` be the type of the consequents in `base_type`.
		3. *If* *UnwrapAffirm:* `Subtype(accessor_type, k)`:
			1. *Return:* `v`.
		4. *Throw:* a new TypeError02 "Type {{ accessor_type }} is not a subtype of type {{ k }}.".
	11. *Throw:* a new TypeError01.
;
```

## Assess
```
Object! Assess(SemanticAccess access) :=
	1. *Assert:* `access.children.count` is 2.
	2. *Let* `base` be `access.children.0`.
	3. *Let* `accessor` be `access.children.1`.
	4. *Let* `base_value` be *Unwrap:* `Assess(base)`.
	5. *If* `accessor` is a SemanticIndex:
		1. *Assert:* `accessor.children.count` is 1.
		2. *Let* `accessor_value` be *Unwrap:* `Assess(accessor.children.0)`.
		3. *Assert:* `accessor_value` is an instance of `Integer`.
		4. *Assert:* `base_value` is an instance of `Tuple` *and* `accessor_value` is an index in `base_value`.
		5. *Return:* the item accessed at index `accessor_value` in `base_value`.
	6. *If* `accessor` is a SemanticKey:
		1. *Let* `id` be `accessor.id`.
		2. *Assert:* `base_value` is an instance of `Record` *and* `id` is a key in `base_value`.
		3. *Return:* the value accessed at key `id` in `base_value`.
	7. *Assert:* `accessor` is a SemanticExpression.
	8. *Let* `accessor_value` be *Unwrap:* `Assess(accessor)`.
	9. *If* `base_value` is an instance of `Tuple`:
		1. *Assert:* `accessor_value` is an instance of `Integer`.
		2. *If* `accessor_value` is an index in `base_value`:
			1. *Return:* the item accessed at index `accessor_value` in `base_value`.
		3. *Throw:* a new VoidError.
	10. *Assert*: `base_value` is an instance of `Mapping`.
	11. *If* `accessor_value` is an antecedent in `base_value`:
		1. Find an antecedent `k` in `base_value` such that `Identical(k, accessor_value)`.
		2. *If* `k` exists:
			1. *Return:* the consequent accessed at antecedent `k` in `base_value`.
	12. *Throw:* a new VoidError.
;
```

# Checklist
- [x] add & document new internal error type `TypeError04`
- [x] add & document new internal error type `VoidError`
- [x] add `.` / access syntax to lexer, parser, and decorate grammars
- [x] variable- and type-check property access expressions
