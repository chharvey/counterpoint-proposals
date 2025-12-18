Type functions are lazily-evaluated, possibly-recursive types that may take type inputs as arguments and return type outputs. This issue covers type function declarations.

Type functions are functions that run at compile-time. They can have zero or more parameters, which are types, and they return a type as an output. Their calls can be used as regular types in regular annotations such as variables, parameters, etc.

## Generic Type Functions
Generic Types formalize the concept of variable types — types that are variable and may change.

A typical example: A generic type function `Nullable<T>` that takes a single type argument `T` and produces a union of `T` and `null`.
```cpl
typefunc Nullable<T> => T | null;
```
`T` is not a type, but a **type variable** that serves as a parameter to the type function. The act of providing an actual value for `T` is called **specifying** the generic type function `Nullable`. It can also be thought of as “calling” or “instantiating” it.

```cpl
let my_value_int: Nullable.<int> = 42;   % resolves to type `int | null`
let my_value_str: Nullable.<str> = null; % resolves to type `str | null`
```
Notice the difference in syntax between generic type declaration and generic type specification: `Nullable<T>` versus `Nullable.<int>`.

Generic type parameters can be reused across type function declarations, in the same way that function parameters can be reused across function declarations.
```cpl
typefunc Nullable<T> => T | null;
typefunc And<T, U>   => T & U;
% parameter `T` is reused
```
Providing the incorrect number of arguments resolves in a TypeError.
```cpl
typefunc Or<T, U> => T | U;
let x: Or.<int> = 42;       %> TypeError: Got 1 generic arguments, but expected 2.
```

Type functions that take inputs are just one example of “generic types” (though not the only one — functions, classes, and interfaces can be generic too). Not all type functions are generic — they could have no parameters. An example of a non-generic type function is:
```cpl
typefunc Number => int | float; % when called, just returns a constant type
```

When considering whether to use a non-generic type function versus a plain type alias, note this difference: Type aliases are injected inline, and evaluated on the spot, whereas type functions are *lazily-evaluated*, meaning they are not evaluated when “called” (“specified”), but rather when they are *assigned* a value.
```cpl
%% 0. %% type TypeAlias = str | [str];
%% 1. %% type A = TypeAlias;           % `TypeAlias` is evaluated here
%% 2. %% let a: A = some_expression;   % type `A` is already known

%% 3. %% type TypeFunction<T> = T | [T];
%% 4. %% type B = TypeFunction.<str>;    % `TypeFunction.<str>` is specified here, but not evaluated yet
%% 5. %% let b: B = some_expression;     % `TypeFunction.<str>` is evaluated here!
```
When a type alias is declared (line 1), it’s evaluated right then and there. Its definition (`str | [str]`) is injected in place wherever it’s used. In other words, type aliases are just a surface-level construct — line 1 is exactly identical to `type A = str | [str];`, and line 2 is exactly identical to `let a: str | [str] = some_expression;`

But on line 4, when a type function is referenced, it’s not evaluated yet! It’s only evaluated, and only part by part, *when assigned an expression*, on line 5. The type-checker first asks, “is `some_expression` assignable to `TypeFunction.<str>`?” and then evaluates `TypeFunction` by parts to compute the answer. This is unlike a type alias where it already knows the type before assigning a variable to it.

This lazy-evaluation is extremely powerful. It means that it’s possible for a type function to never be evaluated! It’s also possible for it to be evaluated several times, returning a different result each time (depending on what’s assigned to it)! We can utilize this mechanism to implement **recursive types**.

## Recursive Type Functions
```cpl
type JsonPrimitive = null | bool | int | float | str;
typefunc JsonValue => JsonPrimitive | List.<JsonValue> | Dict.<JsonValue>;
```
Above is a typical representation of the various JSON types. `JsonValue` cannot be a type alias, since it circularly references itself. When we assign a value to `JsonValue`, the type-checker checks each operand of the type union at assignment-time (during compile-time).

It’s not just unions that are evaluated by parts. Tuple and record types can as well.
```cpl
typefunc BinaryTree => () | (BinaryTree, BinaryTree);
```
In this example, `BinaryTree` references itself, as a 2-tuple of `BinaryTree`s. One might be interested in computing the depth of such an object.
```cpl
function depth(tree: BinaryTree): int
	=> if tree.count > 0
		then 1 + max.(depth.(tree.0), depth.(tree.1))
		else 0;

depth.((
	(),
	( (), () ),
));
```
When `depth` is called, the argument is tested against type `BinaryTree` part by part, until it can no longer be opened up or until it becomes assignable to a part. This means looking into each side of the union and each entry of the tuple.

Type functions can be referenced in type alias definitions, because type aliases are injected inline.
```cpl
type BinaryTreePair = (BinaryTree, BinaryTree);
```
Even though `BinaryTree` is not evaluated yet (because it’s not being assigned a value), the type alias is still valid because the definition `(BinaryTree, BinaryTree)` will get injected wherever `BinaryTreePair` is referenced.

The following example is a recursive generic type function.
```cpl
typefunc Induction<T> => T | Induction.<[T]>;

let x: Induction.<float> = [[42]]; % `Induction.<float>` is evaluated here
```
Since `Induction<T>` is defined as a union, the type-checker tests `[[42]]` on an operand basis — first checking against `float` and then recursively checking against `Induction.<[float]>`, then `[float]`, `Induction.<[[float]]>`, etc.

We can use **type spread** (#68) in type functions.
```cp
typefunc EvenTuple<T> => [] | [T, T, #EvenTuple.<T>];

let n0: EvenTuple.<null> = [];
let n1: EvenTuple.<null> = [null, null];
let n2: EvenTuple.<null> = [null, null, null, null];
let n3: EvenTuple.<null> = [null, null, null, null, null, null];
```

## Infinite Loops
Lazy evaluation can result in infinite loops, which the compiler will report. For example, every possible value is assignable to `typefunc U => U;`. Type `U` is basically equivalent to `anything`; a declaration such as `let x: U = [42];` is valid. Since `U` is a type function, a naïve compiler would evaluate `U` at every step of the way and never stop.

> (“Is `[42]` assignable to `U`? I don’t know, check the definition of `U`.
> Is `[42]` assignable to `U`? I don’t know, …”)

A smarter compiler will memoize variable types. While checking whether `[42]` is assignable to `U`, it assumes *“yes”*, and unless it explicitly returns false, will then use that assumption if it ever comes across such a question again. That way the infinite loop is stopped in its tracks.

> (“Is `[42]` assignable to `U`? I don’t know, but let’s assume so while checking the definition of `U`.
> Is `[42]` assignable to `U`? It appears that it is, so return true.”)

There are other kinds of type functions that aren’t infinite loops, but simply result in unassignable types. For example, no value is ever assignable to `typefunc N => (N,);`. This type declaration is valid, but as soon as an assignment is made (`let x: N = (42,);`), a type error is thrown. Type `N` is basically equivalent to `nothing`.

> (“Is `(42,)` assignable to `N`? I don’t know, but let’s assume so while checking the definition of `N`.
> Is `(42,)` assignable to `(N,)`? Let’s see…
> Is `42` assignable to `N`? I don’t know, but let’s assume so while checking the definition of `N`.
> Is `42` assignable to `(N,)`? It can’t be, since `42` is not a tuple, so return false.”)

## Optional Parameters
Generic type functions can be defined with **optional generic parameters**, which must have a default value. When the generic is specified, the argument may be omitted, in which case the default value is assumed. All optional parameters must come after all required parameters.
```cpl
typefunc Or<T, U? = null> => T | U;
type X = Or.<int, bool>;            % resolves to type `int | bool`
type Y = Or.<int>;                  % resolves to type `int | null`
```

## Constrained Parmeters
Use the `narrows` keyword to constrain a generic parameter. When a parameter `T narrows U` is declared, the argument sent in for `T` must be a subtype of `U`, otherwise it’s a type error.
```cpl
typefunc Nullish<T narrows int | float> => T | null;
type Z = Nullish.<str>;                              %> TypeError: Type `str` is not a subtype of type `int | float`.
```
Use the `widens` keyword to constrain a type parameter in the opposite direction: to declare it as a supertype.
```cpl
typefunc Nullish<T widens int> => T | null;
let x: Nullish.<42 | 43> = 42;              %> TypeError: Type `int` is not a subtype of type `42 | 43`.
```

`widens` is only recommended when `narrows` would require accessing latter parameters.
```cpl
typefunc T<A narrows B, B> => A;
%                    ^ ReferenceError: `B` is used before it is declared.
```
To fix this error, we could switch the parameters:
```cpl
typefunc T<B, A narrows B> => A;
%                       ^ ok
```
However, if switching is not possible, we can use the `widens` keyword:
```cpl
typefunc T<A, B widens A> => A;
%                      ^ ok
```

# Specification

## Lexicon
```diff
Keyword :::=
	// storage
+		| "typefunc"
	// modifier
+		| "narrows"
+		| "widens"
;
```

## Syntax
```diff
+ParameterGeneric<Optional>
+	::= ("_" | IDENTIFIER) <Optional+>"?" (("narrows" | "widens") Type)? <Optional+>("=" Type);

+ParametersGeneric ::=
+	| ","? ParameterGeneric<-Optional># ("," ParameterGeneric<+Optional>#)? ","?
+	| ","?                                   ParameterGeneric<+Optional>#   ","?
+;

+GenericSpecifier
+	::= "<" ParametersGeneric ">";

 DeclarationType         ::= "type"     ("_" | IDENTIFIER)                   "="  Type ";";
+DeclarationTypeFunction ::= "typefunc" ("_" | IDENTIFIER) GenericSpecifier? "=>" Type ";";

Declaration ::=
	| DeclarationType
+	| DeclarationTypeFunction
;
```

## Semantics
```diff
+SemanticHeritage[dir: NARROWS | WIDENS] ::= SemanticType;
+SemanticTypeDefualt                     ::= SemanticType;

+SemanticTypeParam
+	::= SemanticTypeAlias? SemanticHeritage? SemanticDefaultType?;

SemanticTypeCall
	::= SemanticTypeAlias SemanticType*;

-SemanticDeclarationType
+SemanticDeclarationTypeAlias
	::= SemanticTypeAlias? SemanticType;

+SemanticDeclarationTypeFunction
+	::= SemanticTypeAlias? SemanticTypeParam* SemanticType;

SemanticDeclaration =:=
	| SemanticDeclarationVariable
-	| SemanticDeclarationType
+	| SemanticDeclarationTypeAlias
+	| SemanticDeclarationTypeFunction
;
```

## Decorate
```diff
+Decorate(ParameterGeneric<-Optional> ::= "_") -> SemanticTypeParam
+	:= (SemanticTypeParam);
+Decorate(ParameterGeneric<-Optional> ::= IDENTIFIER) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+	);
+Decorate(ParameterGeneric<-Optional> ::= "_" "narrows" Type) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticHeritage[dir=NARROWS] Decorate(Type))
+	);
+Decorate(ParameterGeneric<-Optional> ::= IDENTIFIER "narrows" Type) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		(SemanticHeritage[dir=NARROWS] Decorate(Type))
+	);
+Decorate(ParameterGeneric<-Optional> ::= "_" "widens" Type) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticHeritage[dir=WIDENS] Decorate(Type))
+	);
+Decorate(ParameterGeneric<-Optional> ::= IDENTIFIER "widens" Type) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		(SemanticHeritage[dir=WIDENS] Decorate(Type))
+	);
+Decorate(ParameterGeneric<+Optional> ::= "_" "?" "=" Type) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticDefaultType Decorate(Type))
+	);
+Decorate(ParameterGeneric<+Optional> ::= IDENTIFIER "?" "=" Type) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		(SemanticDefaultType Decorate(Type))
+	);
+Decorate(ParameterGeneric<+Optional> ::= "_" "?" "narrows" Type__0 "=" Type__1) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticHeritage[dir=NARROWS] Decorate(Type__0))
+		(SemanticDefaultType Decorate(Type__1))
+	);
+Decorate(ParameterGeneric<+Optional> ::= IDENTIFIER "?" "narrows" Type__0 "=" Type__1) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		(SemanticHeritage[dir=NARROWS] Decorate(Type__0))
+		(SemanticDefaultType Decorate(Type__1))
+	);
+Decorate(ParameterGeneric<+Optional> ::= "_" "?" "widens" Type__0 "=" Type__1) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticHeritage[dir=WIDENS] Decorate(Type__0))
+		(SemanticTypeDefault Decorate(Type__1))
+	);
+Decorate(ParameterGeneric<+Optional> ::= IDENTIFIER "?" "widens" Type__0 "=" Type__1) -> SemanticTypeParam
+	:= (SemanticTypeParam
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		(SemanticHeritage[dir=WIDENS] Decorate(Type__0))
+		(SemanticTypeDefault Decorate(Type__1))
+	);

+Decorate(ParametersGeneric ::= ParameterGeneric<-Optional># ","?) -> Sequence<SemanticTypeParam>
+	:= ParseList(ParameterGeneric<-Optional>, SemanticTypeParam);
+Decorate(ParametersGeneric ::= ParameterGeneric<+Optional># ","?) -> Sequence<SemanticTypeParam>
+	:= ParseList(ParameterGeneric<+Optional>, SemanticTypeParam);
+Decorate(ParametersGeneric ::= ParameterGeneric<-Optional># "," ParameterGeneric<+Optional># ","?) -> Sequence<SemanticTypeParam>
+	:= [
+		...ParseList(ParameterGeneric<-Optional>),
+		...ParseList(ParameterGeneric<+Optional>),
+	];

+Decorate(GenericSpecifier ::= "<" ParametersGeneric ">") -> Sequence<SemanticTypeParam>
+	:= Decorate(ParametersGeneric);

-Decorate(DeclarationType ::= "type" "_" "=" Type ";") -> SemanticDeclarationType
+Decorate(DeclarationType ::= "type" "_" "=" Type ";") -> SemanticDeclarationTypeAlias
-	:= (SemanticDeclarationType
+	:= (SemanticDeclarationTypeAlias
		Decorate(Type)
	);
-Decorate(DeclarationType ::= "type" IDENTIFIER "=" Type ";") -> SemanticDeclarationType
+Decorate(DeclarationType ::= "type" IDENTIFIER "=" Type ";") -> SemanticDeclarationTypeAlias
-	:= (SemanticDeclarationType
+	:= (SemanticDeclarationTypeAlias
		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);

+Decorate(DeclarationTypeFunction ::= "typefunc" "_" "=>" Type ";") -> SemanticDeclarationTypeFunction
+	:= (SemanticDeclarationTypeFunction
+		Decorate(Type)
+	);
+Decorate(DeclarationTypeFunction ::= "typefunc" IDENTIFIER "=>" Type ";") -> SemanticDeclarationTypeFunction
+	:= (SemanticDeclarationTypeFunction
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);
+Decorate(DeclarationTypeFunction ::= "typefunc" "_" GenericSpecifier? "=>" Type ";") -> SemanticDeclarationTypeFunction
+	:= (SemanticDeclarationTypeFunction
+		...Decorate(GenericSpecifier)
+		Decorate(Type)
+	);
+Decorate(DeclarationTypeFunction ::= "typefunc" IDENTIFIER GenericSpecifier? "=>" Type ";") -> SemanticDeclarationTypeFunction
+	:= (SemanticDeclarationTypeFunction
+		(SemanticTypeAlias[id=TokenWorth(IDENTIFIER)])
+		...Decorate(GenericSpecifier)
+		Decorate(Type)
+	);

-Decorate(Declaration ::= DeclarationType) -> SemanticDeclarationType
+Decorate(Declaration ::= DeclarationType) -> SemanticDeclarationTypeAlias
	:= Decorate(DeclarationType);
+Decorate(Declaration ::= DeclarationTypeFunction) -> SemanticDeclarationTypeFunction
+	:= Decorate(DeclarationTypeFunction);
```
