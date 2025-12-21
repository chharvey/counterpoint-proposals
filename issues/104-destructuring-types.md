Type declarations can be destructured.

# Discussion

## Type Declaration
We can declare many types in one concise destructuring statement:
```cpl
type (A, B) = (int, str);
val x: A = 42;
val y: B = "420";
```

Destructuring applies to nominal type aliases as well.
```cpl
type (X, nominal Y) = (int, int);
val x: X = 42;
val y: Y = 42;        %> TypeError
val y: Y = 42 as <Y>; % ok
```

Above, we used **tuple destructuring**, that is, assigning the “pattern” *(`A`, `B`)* a tuple type. We can also use **record destructuring** by assigning it a record type.
```cpl
type ($A, $B) = (A: int, B: str);
```

As with record punning (#24), the symbol `$` is shorthand for repeating the type name — `($A)` is shorthand for `(A: A)`, where the first `A` is the property in the record type that we’re destructuring, and the second `A` is the new type alias we want to declare. If our record type has different property names, we can use aliases.
```cpl
type (alpha: A, bravo: B) = (bravo: str, alpha: int);
val x: A     = 42;
val y: B     = "420";
val x: alpha = 42;    %> ReferenceError
val y: bravo = "420"; %> ReferenceError
```

Record type destructuring has an advantage over tuple type destructuring: we can change up the order in which we declare types. With a tuple, the order of declared types must match the order of entries in the tuple type. With a record type, we can switch the order as shown above.

Again, we can assign nominal types.
```cpl
type (
	$W,              % punning for `W: W`
	xray: X,
	nominal $Y,      % punning for `Y: nominal Y`
	zulu: nominal Z,
) = (int, int, int, int);
```

### Defaults and Shorthand
As with variable/parameter destructuring (#43, #47), we can provide defaults for symbols that don’t have a corresponding entry.
```cpl
type (A, B? = str)                = (int,);         % `(A, B) == (int, str)`
type ($C? = int, $D? = bool)      = (D: str);       % `(C, D) == (int, str)`
type (echo: E? = int, foxtrot: F) = (foxtrot: str); % `(E, F) == (int, str)`
```

You can use this same notation to short-hand type declarations, if you like.
```cpl
% shorthand for `type (A, B) = (int, str);`
type (A? = int, B? = str) = ();                       % `(A, B) == (int, str)`
type ($C? = int, $D? = str) = (x: bool);              % `(C, D) == (int, str)`
type (echo: E? = int, foxtrot: F? = str) = (x: bool); % `(E, F) == (int, str)`
```

## Nested Destructuring
Destructuring for type declaration can be recursive: We can nest destructured patterns within each other.
```cpl
% regular type destructuring, tuple
type (A, B) = (int, int);

% regular type destructuring, record
type ($C, delta: D) = (c: int, delta: int);

% nested type destructuring, tuple within tuple
type (G, (H, I)) = (int, (int, int));

% nested type destructuring, record within tuple
type (J, ($K, lima: L)) = (int, (K: int, lima: int));

% nested type destructuring, tuple within record
type ($M, november: (N, O)) = (M: int, november: (int, int));

% nested type destructuring, record within record
type ($P, quebec: ($Q, romeo: R)) = (P: int, quebec: (Q: int, romeo: int));
```

## Errors and Caveats

### Type Errors
Type errors are raised when the assigned expression of a destructuring statement doesn’t match the assignee.

Assigning a tuple with greater items is fine, but not fewer items.
```cpl
type (A, B, C) = (int, int, int, int); % index 3 is dropped, but no error
type (D, E, F) = (int, int); %> TypeError (index `2` is missing)
```

Assigning a list gives us the same error.
```cpl
type (D, E, F) = [int]; %> TypeError (`[int]` not assignable to `(anything, anything, anything)`)
```

Assigning a record with extra properties is fine, but not missing properties.
```cpl
type ($a: int, $b: int, $c: int) = (a= 42, b= 420, c= 4200, d= 42000); % `d` is dropped, but no error
type ($d: int, $e: int, $f: int) = (d= 42, e= 420);                    %> TypeError (property `f` is missing)
type (golf= g: int, hotel= h: int) = (golf= 42, h= 420);               %> TypeError (property `hotel` is missing)
```

Of course, the assigned items must be assignable to the variables’ types.
```cpl
val (a, b, c): (int, int, int)       = (42, 420, 123.45);      %> TypeError (`123.45` is not an int)
val ($d: int, echo= e: int) = (d= null, echo= "420"); %> TypeError
```

## Destructuring Generic Type Parameters
Type function parameters can be destructured in the same way as type declarations.

When unrestricted, type parameters only narrow `anything`, which means we cannot infer anything abou them.
```cpl
typefunc Or<T>  => T.0 | T.1; % TypeError: `0` is not a property of type `T`
typefunc And<T> => T.a | T.b; % TypeError: `0` is not a property of type `T`
```
When these generics are “called”, `T` can be specified as any type, so we can’t be sure it has those properties. However, we can restrict `T` with the `narrows` clause.
```cpl
typefunc Or<T narrows (anything, anything)>        => T.0 | T.1;
typefunc And<T narrows (a: anything, b: anything)> => T.0 | T.1;

% specifying type generic must obey destructured pattern
type W = Or.<(int, float)>;        % ok
type X = And.<(b: float, a: int)>; % ok
type Y = Or.<(int,)>;              % TypeError: `(int,)` does not narrow `(anything, anything)`
type Z = And.<(c: int, d: float)>; % TypeError: `(c: int, d: float)` does not narrow `(a: anything, b: anything)`
```

An alternative to providing restrictions with `narrows`, we can destructure the generic parameter `T`.
```cpl
typefunc Or<(U, V)>        => U | V; % `narrows (anything, anything)` is implied
typefunc And<(a: U, b: V)> => U & V; % `narrows (a: anything, b: anything)` is implied
```
By default, each part of the destructured parameter narrows `anything`, but we can still explicitly declare its restriction to refine it more.
```cpl
typefunc Flatten<T narrows ((anything,), (a: anything))>           => (T.0.0, T.1.a); % not destructured
typefunc Flatten<(U, V) narrows ((anything,), (a: anything))>      => (U.0, V.a);     % destructured
typefunc Flatten<(U narrows (anything,), V narrows (a: anything))> => (U.0, V.a);     % destructured, inner style

type W = Flatten.<((int,), (a: float))>; %== (int, float)
```
Above, `U` narrows `(anything,)` and `V` narrows `(a: anything)`, so the property accesses are valid.

However, this can also be achieved with *nested* destructuring.
```cpl
typefunc Flatten<((U), (a: V))> => (U, V);

type W = Flatten.<((int,), (a: float))>; %== (int, float)
```

As with function parameters (#47), we can have *named* destructured generic parameters.
```cpl
typefunc Or<T= (U, V)>        => U | V;
typefunc And<T= (a: U, b: V)> => U & V;

typefunc Flatten<T= (U, V) narrows ((anything,), (a: anything))> => (U.0, V.a);
typefunc Flatten<T= ((U), (a: V))> => (U, V);
```
When specified, the generic type call’s arguments must be labeled.
```cpl
type X = Or.<T= (int, float)>;
type Y = And.<T= (a: int, b: float)>;
type W = Flatten.<T= ((int,), (a: float))>;
```

## Destructuring Generic Type Named Arguments
When *specifying* (“calling”) generic types with named arguments, we can destructure the named arguments in the same way that we can with function calls.
```cpl
type A = Union.<T= int, U= float>;     % not destructured
type B = Union.<(T, U)= (int, float)>; % destructured
```

# Specfication
## Syntax Grammar
```diff
+DestructureTypeAlias<Named, Restricted> ::=
+	| (
+		| <Named+>(Word ":") ("_" | IDENTIFIER)
+		| <Named+>("$" IDENTIFIER)
+	)                                                                      <Restricted+>(("narrows" | "widens") Type)?
+	|               <Named+>(Word ":") DestructureTypeAliases<?Restricted>
+	| <Restricted+>(<Named+>(Word ":") DestructureTypeAliases<-Restricted>              (("narrows" | "widens") Type)?)
+;

+DestructureTypeAliases<Restricted>
+	::= "(" ","? (DestructureTypeAlias<-Named><?Restricted># | DestructureTypeAlias<+Named><?Restricted>#) ","? ")";

ParameterGeneric<Named, Optional> ::=
	| (
		| <Named+>(Word "=") ("_" | IDENTIFIER)
		| <Named+>("$" IDENTIFIER)
	)                                                          <Optional+>"?" (("narrows" | "widens") Type)? <Optional+>("=" Type)
+	| <Named+>(Word "=") DestructureTypeAliases<-Restricted> & <Optional+>"?" (("narrows" | "widens") Type)? <Optional+>("=" Type)
+	| <Named+>(Word "=") DestructureTypeAliases<+Restricted> & <Optional+>"?"                                <Optional+>("=" Type)
;

DeclarationType ::=
	| "type" ("_" | IDENTIFIER)                  (("narrows" | "widens") Type)? "="  Type ";"
+	| "type" DestructureTypeAliases<-Restricted> (("narrows" | "widens") Type)? "="  Type ";"
+	| "type" DestructureTypeAliases<+Restricted>                                "="  Type ";"
;
```
