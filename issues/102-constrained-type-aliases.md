The constraints of generic alias “parameters” is now expanded to type aliases themselves.

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

Type aliases that are constrained by nominal type aliases (#79) must also be nominal. This ensures that nominility is preserved in the type hierarchy.
```cp
type nominal Person = [name: str, dob: nat];
type         Student narrows Person = Person & [studentId: nat]; %> AssignmentError
type nominal Student narrows Person = Person & [studentId: nat]; % fixed

let alice: Student = [name: "Alice", dob: +1991, studentId: +628] as <Student>;
let alice_person: Person = alice; % upcasting allowed because `Student` nominally extends/narrows `Person`

type StudentRecord = Person & [studentId: nat]; % structurally compatible with `Person`, but not a nominal subtype of it

let bob: StudentRecord = [name: "Bob", dob: +1999, studentId: +2718];             %> TypeError: must type-cast to be assignable to `Person` member of intersection
let bob: StudentRecord = [name: "Bob", dob: +1999, studentId: +2718] as <Person>; % ok
let bob_person: Person = bob; %> TypeError % cannot upcast `StudentRecord` to `Person`
```
Since `Person` is nominal, anything assigned to it must nominally be a `Person`. Further, any subtype of it must also be nominal, otherwise we’d be able to assign a structural type to a nominal one through upcasting (as shown in Bob’s example).

# Syntax
```diff
ParameterGeneric<Optional>
	::= IDENTIFIER ("narrows" Type)? <Optional+>("?=" Type);

ParametersGeneric ::=
	|  ParameterGeneric<-Optional># ","?
	| (ParameterGeneric<-Optional># ",")? ParameterGeneric<+Optional># ","?
;

-DeclarationType ::= "type" IDENTIFIER ("<" ","? ParametersGeneric ">"                              )? "=" Type ";";
+DeclarationType ::= "type" IDENTIFIER ("<" ","? ParametersGeneric ">" | ("narrows" | "widens") Type)? "=" Type ";";
```

# Semantics
```diff
SemanticHeritage[dir: NARROWS | WIDENS] ::= SemanticType;

SemanticDeclarationTypeAlias
-	::= SemanticTypeAlias                   SemanticType;
+	::= SemanticTypeAlias SemanticHeritage? SemanticType;

SemanticDeclarationTypeGeneric
	::= SemanticTypeAlias SemanticTypeParam+ SemanticType;
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
```
