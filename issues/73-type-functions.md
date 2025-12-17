Type functions are lazily-evaluated, recursive types. This issue covers type function declarations.

# Discussion
Type functions are functions that run at compile-time, much like like generic type aliases. They can have zero or more parameters, which are types, and they return a type as an output. Their calls can be used as regular types in regular annotations such as variables, parameters, etc.

Unlike type aliases, type functions are *lazily-evaluated*, meaning they are not evaluated when “called” (“specified”), but rather when they are *assigned* a value.
```cp
%% 1. %% type A = TypeAlias;         % `TypeAlias` is evaluated here
%% 2. %% let a: A = some_expression; % type `A` is already known

%% 3. %% type TypeGeneric<T> = T | T[];
%% 4. %% type B = TypeGeneric.<str>;    % `TypeGeneric.<str>` is evaluated here
%% 5. %% let b: B = some_expression;    % type `B` is already known

%% 6. %% typefunc TypeFunction => A & B;
%% 7. %% type C = TypeFunction;
%% 8. %% let c: C = some_expression; % `TypeFunction` is evaluated here!
```
When a non-generic type alias is declared (line 1), it’s evaluated right then and there. When a *generic* type alias is declared (line 3), its definition cannot be evaluated because it has parameters and we didn’t provide any arguments. But when it’s **specified** (provided with arguments, line 4), it’s evaluated at that point.

Like a generic type alias, a **type function** isn’t evaluated when declared. But wait, even on line 7, when `TypeFunction` is referenced, it still isn’t evaluated yet! The type is only evaluated, and only part by part, *when assigned an expression*, on line 8. The type-checker first asks, “is `some_expression` assignable to `TypeFunction`?” and then evaluates `TypeFunction` by parts to compute the answer. This is unlike a type alias where it already knows the type before assigning a variable to it.

This lazy-evaluation is extremely powerful. It means that it’s possible for a type function to never be evaluated! It’s also possible for it to be evaluated several times, returning a different result each time (depending on what’s assigned to it)! We can utilize this mechanism to implement **recursive types**.

## Recursive Type Functions
```cp
type JsonPrimitive = null | bool | int | float | str;
typefunc JsonValue => JsonPrimitive | JsonValue[] | Record.<JsonValue>;
```
Above is a typical representation of the various JSON types. `JsonValue` cannot be a type alias, since it circularly references itself. When we assign a value to `JsonValue`, the type-checker checks each operand of the type union at assignment-time (during compile-time).

It’s not just unions that are evaluated by parts. Tuple and record types can as well.
```cp
typefunc BinaryTree => [] | [BinaryTree, BinaryTree];
```
In this example, `BinaryTree` references itself, as a 2-tuple of `BinaryTree`s. One might be interested in computing the depth of such an object.
```cp
function depth(tree: BinaryTree): int
	=> if tree.count > 0
		then 1 + max.(depth.(tree.0), depth.(tree.1))
		else 0;

depth.([
	[],
	[ [], [] ],
]);
```
When `depth` is called, the argument is tested against type `BinaryTree` part by part, until it can no longer be opened up or until it becomes assignable to a part. This means looking into each side of the union and each entry of the tuple.

Type functions *cannot* be referenced in type alias definitions (even generic ones).
```cp
type BinaryTreePair = [BinaryTree, BinaryTree]; %> TypeError
```
> TypeError: Cannot call type function `BinaryTree` in type alias definition.

Since type functions are lazily evaluated, it’s impossible for the compiler to evaluate `BinaryTreePair` as it does all type aliases, at declaration time. We could fix this by making `BinaryTreePair` itself a type function. And of course, we can always reference any type, function or not, within the definition of a type function.

## Generic Type Functions
The following example is a recursive generic type function.
```cp
typefunc Induction<T> => T | Induction.<[T]>;

let x: Induction.<float> = [[42]]; % `Induction.<float>` is evaluated here
```
Since `Induction<T>` is defined as a union, the type-checker tests `[[42]]` on an operand basis — first checking against `float` and then recursively checking against `Induction<[float]>`, then `[float]`, `Induction<[[float]]>`, etc.

We can use **type spread** (#68) in type functions.
```cp
typefunc EvenTuple<T> => [] | [T, T, #EvenTuple.<T>];

let n0: EvenTuple.<null> = [];
let n1: EvenTuple.<null> = [null, null];
let n2: EvenTuple.<null> = [null, null, null, null];
let n3: EvenTuple.<null> = [null, null, null, null, null, null];
```

## Infinite Loops
Lazy evaluation can result in infinite loops, which the compiler will report. For example, every possible value is assignable to `typefunc U => U;`. Type `U` is basically equivalent to `unknown`; a declaration such as `let x: U = [42];` is valid. Since `U` is a type function, a naïve compiler would evaluate `U` at every step of the way and never stop.

> (“Is `[42]` assignable to `U`? I don’t know, check the definition of `U`.
> Is `[42]` assignable to `U`? I don’t know, …”)

A smarter compiler will memoize variable types. While checking whether `[42]` is assignable to `U`, it assumes *“yes”*, and unless it explicitly returns false, will then use that assumption if it ever comes across such a question again. That way the infinite loop is stopped in its tracks.

> (“Is `[42]` assignable to `U`? I don’t know, but let’s assume so while checking the definition of `U`.
> Is `[42]` assignable to `U`? It appears that it is, so return true.”)

There are other kinds of type functions that aren’t infinite loops, but simply result in unassignable types. For example, no value is ever assignable to `typefunc N => [N];`. This type declaration is valid, but as soon as an assignment is made (`let x: N = [42];`), a type error is thrown. Type `N` is basically equivalent to `never`.

# Specification

## Lexicon
```diff
Keyword :::=
	// storage
+		| "typefunc"
;
```

## Syntax
```diff
 DeclarationType         ::= "type"     ("_" | IDENTIFIER) GenericSpecifier? "="  Type ";";
+DeclarationTypeFunction ::= "typefunc" ("_" | IDENTIFIER) GenericSpecifier? "=>" Type ";";

Declaration ::=
	| DeclarationType
+	| DeclarationTypeFunction
;
```

## Semantics
```diff
SemanticTypeCall
	::= SemanticTypeAlias SemanticType*;

SemanticDeclarationTypeAlias
	::= SemanticTypeAlias? SemanticType;

SemanticDeclarationTypeGeneric
	::= SemanticTypeAlias? SemanticTypeParam+ SemanticType;

+SemanticDeclarationTypeFunction
+	::= SemanticTypeAlias? SemanticTypeParam* SemanticType;

SemanticDeclaration =:=
	| SemanticDeclarationVariable
	| SemanticDeclarationTypeAlias
	| SemanticDeclarationTypeGeneric
+	| SemanticDeclarationTypeFunction
;
```

## Decorate
```diff
+Decorate(GenericSpecifier ::= "<" ParametersGeneric ">") -> Sequence<SemanticTypeParam>
+	:= Decorate(ParametersGeneric);

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

+Decorate(Declaration ::= DeclarationTypeFunction) -> SemanticDeclarationTypeFunction
+	:= Decorate(DeclarationTypeFunction);
```
