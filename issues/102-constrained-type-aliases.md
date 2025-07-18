The constraints of generic alias “parameters” is now expanded to type aliases themselves. Follows #66.

Optionally, a type alias may be declared with a `narrows` or `widens` constraint to further stricten the declaration.
```cp
type JustInt narrows Maybe.<int> = int;
```

This constraint clause is optional, but helps catch errors early on. The following change is valid, but could be prone to more bugs:
```diff
-type JustInt = int;
+type JustInt = float;
```
With the constraint in place, the typer throws a compile-time error in one place — at the type alias’s declaration site — rather than at all its reference sites.
```cp
type JustInt narrows Maybe.<int> = float; %> TypeError: `float` is not a subtype of `Maybe.<int>`
```

Note that type alias constraints do not (and cannot) apply to generic type aliases.
```cp
type MyType<T> narrows T | null = SomeComplicatedGeneric.<T> | T; %> SyntaxError
```
The reason is that there’s just no way, in the general case and at the declaration site, for the compiler to evaluate the assigned type expression before it’s specified with a type argument.

Also note that a constrained type alias may be generic and have a parameter that is itself constrained.
```cp
type T<S narrows int> widens float = float | S; % allowed because `float | S` widens `float`
type Example = T.<42>; %== float | 42
```


# Syntax
```diff
ParameterGeneric<Optional>
	::= IDENTIFIER ("narrows" Type)? <Optional+>("?=" Type);

ParametersGeneric ::=
	|  ParameterGeneric<-Optional># ","?
	| (ParameterGeneric<-Optional># ",")? ParameterGeneric<+Optional># ","?
;

-DeclarationType ::= "type" IDENTIFIER ("<" ","? ParametersGeneric ">")?                                "=" Type ";";
+DeclarationType ::= "type" IDENTIFIER ("<" ","? ParametersGeneric ">")? (("narrows" | "widens") Type)? "=" Type ";";
```

# Semantics
```diff
SemanticHeritage[dir: NARROWS | WIDENS] ::= SemanticType;

SemanticDeclarationTypeAlias
-	::= SemanticTypeAlias                   SemanticType;
+	::= SemanticTypeAlias SemanticHeritage? SemanticType;

SemanticDeclarationTypeGeneric
-	::= SemanticTypeAlias SemanticTypeParam+                   SemanticType;
+	::= SemanticTypeAlias SemanticTypeParam+ SemanticHeritage? SemanticType;
```

# Decorate
```diff
Decorate(DeclarationType ::= "type" IDENTIFIER "=" Type ";") -> SemanticDeclarationTypeAlias
	:= (SemanticDeclarationTypeAlias
		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
+Decorate(DeclarationType ::= "type" IDENTIFIER "narrows" Type__0 "=" Type__1 ";") -> SemanticDeclarationTypeAlias
+	:= (SemanticDeclarationTypeAlias
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		(SemanticHeritage[dir=NARROWS] Decorate(Type__0))
+		Decorate(Type__1)
+	);
+Decorate(DeclarationType ::= "type" IDENTIFIER "widens" Type__0 "=" Type__1 ";") -> SemanticDeclarationTypeAlias
+	:= (SemanticDeclarationTypeAlias
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		(SemanticHeritage[dir=WIDENS] Decorate(Type__0))
+		Decorate(Type__1)
+	);
Decorate(DeclarationType ::= "type" IDENTIFIER "<" ","? ParametersGeneric ">" "=" Type ";") -> SemanticDeclarationTypeGeneric
	:= (SemanticDeclarationTypeGeneric
		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
		...Decorate(ParametersGeneric)
		Decorate(Type)
	);
+Decorate(DeclarationType ::= "type" IDENTIFIER "<" ","? ParametersGeneric ">" "narrows" Type__0 "=" Type__1 ";") -> SemanticDeclarationTypeGeneric
+	:= (SemanticDeclarationTypeGeneric
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		...Decorate(ParametersGeneric)
+		(SemanticHeritage[dir=NARROWS] Decorate(Type__0))
+		Decorate(Type__1)
+	);
+Decorate(DeclarationType ::= "type" IDENTIFIER "<" ","? ParametersGeneric ">" "widens" Type__0 "=" Type__1 ";") -> SemanticDeclarationTypeGeneric
+	:= (SemanticDeclarationTypeGeneric
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		...Decorate(ParametersGeneric)
+		(SemanticHeritage[dir=WIDENS] Decorate(Type__0))
+		Decorate(Type__1)
+	);
```
