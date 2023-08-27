Type `void` and nullish type operator change.

# Discussion

## The `void` Type
Introducting a new type: Type `void` represents the absense of a value. `void` is unusual in that it doesn’t exactly represent a set of values like the other types do.

`void` is similar to `never` in that there are no values assignable to it, but there are major differences between the two types. Type `never` is the bottom type, and it represents an expression that never evaluates or a procedure that never executes. `never` is the “Identity element of Union”: `T | never` is exactly `T`. It’s also the “Absorption element of Intersection”: `T & never` is exactly `never`. The bottom type is unique in that it’s a subtype of every other type. On the other hand, `void` is a strict supertype of `never`. Unioning a type with void expressly produces a new type: `T | void` is almost always different from `T`. The intersection `T & void` is not particularly useful, but nevertheless it’s not the same as `void`. And `void` is not a subtype of any of the built-in types — it lives amongst the other primitive types such as `int` and `bool`.

`void` is also unlike `null`. Type `null` represents a placeholder value that has no prescribed semantics, but it’s still a value nonetheless. Type `void` has no values assignable to it whatsoever. **It represents the completion of an evaluation, but the absence of a value.**

## Breaking: The Nullish Type Operator
The Nullish type operator is shorthand syntax for unioning a type with `null`. In previous versions, this postfix unary operator used the punctuator `!` (U+0021 EXCLAMATION MARK). From now on, the punctuator `?` (U+003F QUESTION MARK) is the designated symbol. The exclamation mark is now a semantic error, but is still reserved for a different type operation (see #69). **This is a breaking change.**
```cp
type IntOrNull = int?;       % int | null
type T         = int!;       % TypeError for now, but will be used later for something else
type IntOrVoid = int | void; % there’s no designated shorthand for this
```

# Specification

## Lexicon
```diff
Keyword :::=
	// literal
+		| "void"
;
```

## Syntax
```diff
TypeKeyword ::=
+	| "void"
	| "bool"
	| "int"
	| "float"
	| "str"
;

TypeUnarySymbol ::=
	| TypeUnit
+	| TypeUnarySymbol "?"
	| TypeUnarySymbol "!"
;
```

## Semantics
```diff
+SemanticTypeOperation[operator: OREXCP]
+	::= SemanticType;
```

## Decorate
```diff
+Decorate(TypeKeyword ::= "void") -> SemanticTypeConstant
+	:= (SemanticTypeConstant[value=Void]);

+Decorate(TypeUnarySymbol ::= TypeUnarySymbol "?") -> SemanticTypeOperation
+	:= (SemanticTypeOperation[operator=ORNULL]
+		Decorate(TypeUnarySymbol)
+	);
Decorate(TypeUnarySymbol ::= TypeUnarySymbol "!") -> SemanticTypeOperation
-	:= (SemanticTypeOperation[operator=ORNULL]
+	:= (SemanticTypeOperation[operator=OREXCP]
+		Decorate(TypeUnarySymbol)
+	);
```

## Assess
```diff
Type Assess(SemanticTypeOperation[operator: ORNULL] oper) :=
	1. *Assert:* `oper.children.count` is 1.
	2. *Let* `child` be *UnwrapAffirm:* `Assess(oper.children.0)`.
	3. *Return:* `Union(child, Null)`.
;

+Type Assess(SemanticTypeOperation[operator: OREXCP] oper) :=
+	1. *Throw:* new TypeError "Operator not yet supported.".
+;
```

# Checklist
- [x] use `?` for nullish type operator & throw TypeError for `!`
- [x] add new Void type to type system
- [x] add `void` keyword to lexer & parser grammars
- [x] update spec & reference documentation
