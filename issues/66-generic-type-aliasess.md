Generic Types formalize the concept of variable types — types that are variable and may change.

# Discussion
A typical example: A generic type `Nullable<T>` that takes a single type argument `T` and produces a union of `T` and `null`.
```cp
type Nullable<T> = T | null;
```
`T` is not a type, but a **type variable** that acts like a parameter. The act of providing an actual value for `T` is called **specifying** the generic type `Nullable`. It can also be thought of as “calling” or “instantiating” it.
```cp
let my_value_int: Nullable.<int> = 42;   % resolves to type `int | null`
let my_value_str: Nullable.<str> = null; % resolves to type `str | null`
```
Notice the difference in syntax between generic type declaration and generic type specification: `Nullable<T>` versus `Nullable.<int>`.

Generic type parameters can be reused across type declarations.
```cp
type Nullable<T> = T | null;
type And<T, U>  = T & U;
% parameter `T` is reused
```

Providing the incorrect number of arguments resolves in a TypeError.
```cp
type Or<T, U> = T | U;
let x: Or.<int> = 42;  %> TypeError: Got 1 type arguments, but expected 2.
```

Generic type aliases can be thought of as “functions” that have “parameters” and are called with “arguments”, however this is not exactly accurate, since **type functions** are a related but separate feature (#73). Type generics are resolved eagerly, and like non-generics, they cannot be recursive.
```cp
type Or<A, B> = A | Or.<B, null>; %> ReferenceError: `Or` is not defined
```

## Optional Parameters
Generic type aliases can be defined with **optional generic parameters**, which must have a default value. When the generic is specified, the argument may be omitted, in which case the default value is assumed. All optional parameters must come after all required parameters.
```cp
type Or<T, U ?= null> = T | U;
type X = Or.<int, bool>;       % resolves to type `int | bool`
type Y = Or.<int>;             % resolves to type `int | null`
```

## Constrained Parmeters
Use the `narrows` keyword to constrain a generic parameter. When a parameter `T narrows U` is declared, the argument sent in for `T` must be a subtype of `U`, otherwise it’s a type error.
```cp
type Nullish<T narrows int | float> = T | null;
type Z = Nullish.<str>;                         %> TypeError: Type `str` is not a subtype of type `int | float`.
```

# Specification

## Lexicon
```diff
Keyword :::=
	// modifier
+		| "narrows"
;
```

## Syntax
```diff
+ParameterGeneric<Optional>
+	::= IDENTIFIER ("narrows" Type)? <Optional+>("?=" Type);

+ParametersGeneric ::=
+	|  ParameterGeneric<-Optional># ","?
+	| (ParameterGeneric<-Optional># ",")? ParameterGeneric<+Optional># ","?
+;

+GenericSpecifier
+	::= "<" ","? ParametersGeneric ">";

-DeclarationType ::= "type" IDENTIFIER                   "=" Type ";";
+DeclarationType ::= "type" IDENTIFIER GenericSpecifier? "=" Type ";";
```

## Semantics
```diff
+SemanticHeritage    ::= SemanticType;
+SemanticDefaultType ::= SemanticType;

+SemanticTypeParam
+	::= SemanticTypeAlias SemanticHeritage? SemanticDefaultType?;

-SemanticDeclarationType
+SemanticDeclarationTypeAlias
	::= SemanticTypeAlias SemanticType;

+SemanticDeclarationTypeGeneric
+	::= SemanticTypeAlias SemanticTypeParam+ SemanticType;

SemanticDeclaration =:=
	| SemanticDeclarationVariable
-	| SemanticDeclarationType
+	| SemanticDeclarationTypeAlias
+	| SemanticDeclarationTypeGeneric
;
```

## Decorate
```diff
+Decorate(ParameterGeneric<-Optional> ::= IDENTIFIER) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+	);
+Decorate(ParameterGeneric<-Optional> ::= IDENTIFIER "narrows" Type) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		(SemanticHeritage Decorate(Type))
+	);
+Decorate(ParameterGeneric<+Optional> ::= IDENTIFIER "?=" Type) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		(SemanticDefaultType Decorate(Type))
+	);
+Decorate(ParameterGeneric<+Optional> ::= IDENTIFIER "narrows" Type__0 "?=" Type__1) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		(SemanticHeritage Decorate(Type__0))
+		(SemanticDefaultType Decorate(Type__1))
+	);

+Decorate(ParametersGeneric ::= ParameterGeneric<?Optional># ","?) -> Sequence<SemanticTypeParam>
+	:= ParseList(ParameterGeneric<?Optional>, SemanticTypeParam);
+Decorate(ParametersGeneric ::= ParameterGeneric<-Optional># "," ParameterGeneric<+Optional># ","?) -> Sequence<SemanticTypeParam>
+	:= [
+		...ParseList(ParameterGeneric<-Optional>),
+		...ParseList(ParameterGeneric<+Optional>),
+	];

-Decorate(DeclarationType ::= "type" IDENTIFIER "=" Type ";") -> SemanticDeclarationType
+Decorate(DeclarationType ::= "type" IDENTIFIER "=" Type ";") -> SemanticDeclarationTypeAlias
-	:= (SemanticDeclarationType
+	:= (SemanticDeclarationTypeAlias
		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
+Decorate(DeclarationType ::= "type" IDENTIFIER "<" ","? ParametersGeneric ">" "=" Type ";") -> SemanticDeclarationTypeGeneric
+	:= (SemanticDeclarationTypeGeneric
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		...Decorate(ParametersGeneric)
+		Decorate(Type)
+	);

-Decorate(Declaration ::= DeclarationType) -> SemanticDeclarationType
+Decorate(Declaration ::= DeclarationType) -> SemanticDeclarationTypeAlias | SemanticDeclarationTypeGeneric
	:= Decorate(DeclarationType);
```
