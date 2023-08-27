Assignment of mutable data structures’ entries. Depends on #25.

```cp
let tuple: mutable [str, str] = ["hello", "world"];
tuple.[3 - 2] = "mundo";
tuple == ["hello", "mundo"]; % true

let record: mutable obj = [];
record.[[x= 5, y= 3]] = 2;
record == [
	[y= 3, x= 5] |-> 2,
]; % true
```

*DRAFT: Using atoms (#59) as record keys:*
```cp
type Font = [
	weight: int,
	size:   float,
	style:  .BOLD | .ITALIC | .OBLIQUE,
];
let font: mutable Font = [
	weight= 400,
	size=   12.0,
	style=  .ITALIC,
];
font.[.weight] = 700;
let sz: .size = .size;
font.[sz] = 18.5;
font.[if font.size < 24 then .style else .sTyLe] = .OBLIQUE;
```

## Invariance
Tuple and record types that are mutable are **invariant** as supertypes, meaning for any types ‹S› and ‹T›, if ‹S› is a subtype of ‹T›, then it is never the case that ‹G›<‹S›> is a subtype of ‹G›<​mutable ‹T›>. This invariance ensures type safety for mutating operations.
```cp
let sub: mutable [int, int] = [4, 2];
let 'super': mutable [int?, int?] = sub; %> TypeError
```
> TypeError: Expression of type `mutable [int, int]` is not assignable to type `mutable [int?, int?]`.

We get a type error, even though `int` is assignable to `int?`. Without this type safety, we’d be able to mutate `'super'` in a way that would otherwise be invalid for `sub`:
```cp
'super'.0 = null;
```
The type checker wouldn’t see a problem with this, but by setting `'super'.0 = null` we would be actually setting `sub.0 = null` at runtime (since `'super'` points to `sub`), and that’s invalid because `sub` cannot contain `null`.

However, *immutable* tuple and record types remain **covariant** as discussed in #53.
```cp
let sub: mutable [int, int] = [4, 2];
let 'super': [int?, int?] = sub; % no error
```
Type `mutable [int, int]` is a subtype of type `[int?, int?]`, even if the former is mutable.

# Specification

## Syntactic Grammar
```diff
TypeUnarySymbol   ::= TypeUnit        | TypeUnarySymbol "?";
+TypeUnaryKeyword ::= TypeUnarySymbol | "mutable" TypeUnaryKeyword;

-TypeIntersection ::= (TypeIntersection "&")? TypeUnarySymbol;
+TypeIntersection ::= (TypeIntersection "&")? TypeUnaryKeyword;
TypeUnion         ::= (TypeUnion        "|")? TypeIntersection;

ExpressionCompound ::=
	| ExpressionUnit
	| ExpressionCompound PropertyAccess
;

Assignee ::=
	| IDENTIFIER
+	| ExpressionCompound PropertyAccess
;

StatementAssignment
	::= Assignee "=" Expression ";";
```

## Semantic Schema
```diff
+SemanticTypeOperation[operator: MUTABLE]
+	::= SemanticType;

SemanticAccess
	::= SemanticExpression (SemanticIndex | SemanticKey | SemanticExpression);

SemanticAssignment
-	::=  SemanticVariable                   SemanticExpression;
+	::= (SemanticVariable | SemanticAccess) SemanticExpression;
```

## Decorate
```diff
+Decorate(TypeUnaryKeyword ::= TypeUnarySymbol) -> SemanticType
+	:= Decorate(TypeUnarySymbol);
+Decorate(TypeUnaryKeyword ::= "mutable" TypeUnaryKeyword) -> SemanticTypeOperation
+	:= (SemanticTypeOperation[operator=MUTABLE]
+		Decorate(TypeUnaryKeyword)
+	);

-Decorate(TypeIntersection ::= TypeUnarySymbol) -> SemanticType
-	:= Decorate(TypeUnarySymbol);
+Decorate(TypeIntersection ::= TypeUnaryKeyword) -> SemanticType
+	:= Decorate(TypeUnaryKeyword);


Decorate(ExpressionCompound ::= ExpressionUnit) -> SemanticExpression
	:= Decorate(ExpressionUnit);
Decorate(ExpressionCompound ::= ExpressionCompound PropertyAccess) -> SemanticAccess
	:= (SemanticAccess
		Decorate(ExpressionCompound)
		Decorate(PropertyAccess)
	);

Decorate(Assignee ::= IDENTIFIER) -> SemanticVariable
	:= (SemanticVariable[id=TokenWorth(IDENTIFIER)]);
+Decorate(Assignee ::= ExpressionCompound PropertyAccess) -> SemanticAccess
+	:= (SemanticAccess
+		Decorate(ExpressionCompound)
+		Decorate(PropertyAccess)
+	);

Decorate(StatementAssignment ::= Assignee "=" Expression ";") -> SemanticAssignment
	:= (SemanticAssignment
		Decorate(Assignee)
		Decorate(Expression)
	);
```

# Checklist
- [x] add & document error type `MutabilityError`
- [x] update subtyping & variance rules
- [x] lex, parse, & decorate `mutable` type operator
- [x] lex, parse, & decorate property assignment statements
- [x] update var/type-checking of assignment statements
