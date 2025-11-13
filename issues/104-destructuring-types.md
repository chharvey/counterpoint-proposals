Type declarations can be destructured.

# Discussion

## Type Declaration
We can declare many types in one concise destructuring statement:
```cp
type [A, B] = [int, str];
let x: A = 42;
let y: B = "420";
```

Destructuring applies to nominal type aliases as well.
```cp
type [X, nominal Y] = int[2];
let x: X = 42;
let y: Y = 42;        %> TypeError
let y: Y = 42 as <Y>; % ok
```

Above, we used **tuple destructuring**, that is, assigning the “pattern” *[`A`, `B`]* a tuple type. We can also use **record destructuring** by assigning it a record type.
```cp
type [$A, $B] = [A: int, B: str];
```

As with record punning (#24), the symbol `$` is shorthand for repeating the type name — `[$A]` is shorthand for `[A: A]`, where the first `A` is the property in the record type that we’re destructuring, and the second `A` is the new type alias we want to declare. If our record type has different property names, we can use aliases.
```cp
type [alpha: A, bravo: B] = [bravo: str, alpha: int];
let x: A     = 42;
let y: B     = "420";
let x: alpha = 42;    %> ReferenceError
let y: bravo = "420"; %> ReferenceError
```

Record type destructuring has an advantage over tuple type destructuring: we can change up the order in which we declare types. With a tuple, the order of declared types must match the order of entries in the tuple type. With a record type, we can switch the order as shown above.

Again, we can assign nominal types.
```cp
type [
	$W,              % punning for `W: W`
	xray: X,
	nominal $Y,      % punning for `Y: nominal Y`
	zulu: nominal Z,
] = int[4];
```

### Defaults and Shorthand
As with variable/parameter destructuring (#43, #47), we can provide defaults for symbols that don’t have a corresponding entry.
```cp
type [A, B ?= str]                = [int];          % `[A, B] == [int, str]`
type [$C ?= int, $D ?= bool]      = [D: str];       % `[C, D] == [int, str]`
type [echo: E ?= int, foxtrot: F] = [foxtrot: str]; % `[E, F] == [int, str]`
```

You can use this same notation to short-hand type declarations, if you like.
```cp
% shorthand for `type [A, B] = [int, str];`
type [A ?= int, B ?= str] = [];                       % `[A, B] == [int, str]`
type [$C ?= int, $D ?= str] = [x: bool];              % `[C, D] == [int, str]`
type [echo: E ?= int, foxtrot: F ?= str] = [x: bool]; % `[E, F] == [int, str]`
```

## Nested Destructuring
Destructuring for type declaration can be recursive: We can nest destructured patterns within each other.
```cp
% regular type destructuring, tuple
type [A, B] = int[2];

% regular type destructuring, record
type [$C, delta: D] = [c: int, delta: int];

% nested type destructuring, tuple within tuple
type [G, [H, I]] = [int, int[2]];

% nested type destructuring, record within tuple
type [J, [$K, lima: L]] = [int, [K: int, lima: int]];

% nested type destructuring, tuple within record
type [$M, november: [N, O]] = [M: int, november: int[2]];

% nested type destructuring, record within record
type [$P, quebec: [$Q, romeo: R]] = [P: int, quebec: [Q: int, romeo: int]];
```

## Generic Parameters
Type aliases with generic parameters may be declared in destructuring patterns.
```cp
type [A<T>, nominal B<U= V>] = [int | T, float & V];
type [$C, d: nominal D<out W narrows [unknown] ?= X>] = [C: B.<U= null>, d: W.0];

typefunc A<T> => int | T;
```

## Errors and Caveats

### Type Errors
Type errors are raised when the assigned expression of a destructuring statement doesn’t match the assignee.

Assigning a tuple with greater items is fine, but not fewer items.
```cp
type [A, B, C] = int[4]; % index 3 is dropped, but no error
type [D, E, F] = int[2]; %> TypeError (index `2` is missing)
```

Assigning a list gives us the same error.
```cp
type [D, E, F] = int[]; %> TypeError (`int[]` not assignable to `unknown[3]`)
```

Assigning a record with extra properties is fine, but not missing properties.
```cp
type [$a: int, $b: int, $c: int] = [a= 42, b= 420, c= 4200, d= 42000]; % `d` is dropped, but no error
type [$d: int, $e: int, $f: int] = [d= 42, e= 420];                    %> TypeError (property `f` is missing)
type [golf= g: int, hotel= h: int] = [golf= 42, h= 420];               %> TypeError (property `hotel` is missing)
```

Of course, the assigned items must be assignable to the variables’ types.
```cp
let [a, b, c]: int[3]       = [42, 420, 123.45];      %> TypeError (`123.45` is not an int)
let [$d: int, echo= e: int] = [d= null, echo= "420"]; %> TypeError
```

## Destructuring Generic Type Parameters
Type alias parameters and type function parameters can be destructured in the same way as type declarations.

When unrestricted, type parameters only narrow `unknown`, which means we cannot infer anything abou them.
```cp
type Or<T>  = T.0 | T.1; % TypeError: `0` is not a property of type `T`
type And<T> = T.a | T.b; % TypeError: `0` is not a property of type `T`
```
When these generics are “called”, `T` can be specified as any type, so we can’t be sure it has those properties. However, we can restrict `T` with the `narrows` clause.
```cp
type Or<T narrows [unknown, unknown]>        = T.0 | T.1;
type And<T narrows [a: unknown, b: unknown]> = T.0 | T.1;

% specifying type generic must obey destructured pattern
type W = Or.<[int, float]>;        % ok
type X = And.<[b: float, a: int]>; % ok
type Y = Or.<[int]>;               % TypeError: `[int]` does not narrow `[unknown, unknown]`
type Z = And.<[c: int, d: float]>; % TypeError: `[c: int, d: float]` does not narrow `[a: unknown, b: unknown]`
```

An alternative to providing restrictions with `narrows`, we can destructure the parameter `T`.
```cp
type Or<T= [U, V]>        = U | V;                 % `narrows [unknown, unknown]` is implied
type And<T= [a: U, b: V]> = U & V;                 % `narrows [a: unknown, b: unknown]` is implied
```
By default, each part of the destructured parameter narrows `unknown`, but we can still explicitly declare its restriction to refine it more.
```cp
type Flatten<T narrows [[unknown], [a: unknown]]>         = [T.0.0, T.1.a]; % not destructured
type Flatten<T= [U, V] narrows [[unknown], [a: unknown]]> = [U.0, V.a];     % destructured

type W = Flatten.<[[int], [a: float]]>; %== [int, float]
```
Above, `U` narrows `[unknown]` and `V` narrows `[a: unknown]`, so the property accesses are valid.

However, this can also be achieved with *nested* destructuring.
```cp
type Flatten<T= [[U], [a: V]]> = [U, V];

type W = Flatten.<[[int], [a: float]]>; %== [int, float]
```

## Destructuring Record Type Properties
Similar to destructuring properties of records (#44), we can destructure the properties of record *types*:
```cp
type R = [
	a:      int,
	[b, c]: str[2], % shorthand for `b: str, c: str,`
];
```

## Destructuring Generic Type Named Arguments
When *specifying* (“calling”) generic types with named arguments, we can destructure the named arguments in the same way that we can with function calls.
```cp
type A = Or.<T= int, U= float>;     % not destructured
type B = Or.<[T, U]= [int, float]>; % destructured
```

# Specfication
## Syntax Grammar
```diff
ParameterGeneric<Optional>
-	::= (Word "=")?  IDENTIFIER                           (("narrows" | "widens") Type)? <Optional+>("?=" Type);
+	::= (Word "=")? (IDENTIFIER | DestructureTypeAliases) (("narrows" | "widens") Type)? <Optional+>("?=" Type);

ParametersGeneric ::=
	|  ParameterGeneric<-Optional># ","?
	| (ParameterGeneric<-Optional># ",")? ParameterGeneric<+Optional># ","?
;

GenericSpecifier
	::= "<" ","? ParametersGeneric ">";


+DestructureTypeAliasItem ::=
+	| IDENTIFIER
+	| DestructureTypeAliases
+;

+DestructureTypeAliasKey ::=
+	| "$" IDENTIFIER
+	| Word ":" DestructureTypeAliasItem
+;

+DestructureTypeAliases ::=
+	| "[" ","? DestructureTypeAliasItem# ","? "]"
+	| "[" ","? DestructureTypeAliasKey#  ","? "]"
+;

-DeclarationType         ::= "type"      IDENTIFIER GenericSpecifier?                           "="  Type ";";
+DeclarationType         ::= "type"     (IDENTIFIER GenericSpecifier? | DestructureTypeAliases) "="  Type ";";
 DeclarationTypeFunction ::= "typefunc"  IDENTIFIER GenericSpecifier?                           "=>" Type ";";
```
