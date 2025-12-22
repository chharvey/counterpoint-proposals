Like function parameters (#46), type parameters (of a type alias or type function) may be named. The name may be supplied when **specified** (“called”).

```cpl
typefunc Union<Left= A, Right= B> => A | B;
typefunc Intersection<Left= A, Right= B? = null> => A & B;

type IntOrFloat = Union.<Left=  int,   Right= float>; %== int   | float
type IntOrFloat = Union.<Right= float, Left=  int>;   %== int   | float
type FloatOrInt = Union.<float,        Right= int>;   %== float | int
type FloatOrInt = Union.<Left= float,  int>;          %> ParseError

type IntAndNull = Intersection.<Left= int>;  %== int & null
type IntAndNull = Intersection.<int>;        %== int & null
type IntAndNull = Intersection.<Right= int>; %> TypeError % expected an argument for parameter `Left`
```
Identifiers `A` and `B` are **internal**: accessible at the “define site”; identifiers `Left` and `Right` are **external**: accessible at the “call site”.

We can use `$`-punning for shorthand when the external and internal name are the same: `$T` is shorthand for `T= T`.
```cpl
typefunc Union<$A, $B> => A | B;
type IntOrFloat = Union.<A= int, B= float>;
```

The same rules of named arguments (#57) apply:

- All positional arguments must come before all named arguments.
- All positional arguments are assigned first, followed by all named arguments.
- All argument values are evaluated in source order.
- After all given arguments are assigned, any unspecified optional parameters receive their default value (#66).

## Syntax
```diff
+GenericArgumentNamed ::=
+	| "$" IDENTIFIER
+	| Word "=" Type
+;

GenericArguments ::=
-	| "<" ","? Type#                              ","? ">"
+	| "<" ","? Type# ("," GenericArgumentNamed#)? ","? ">"
+	| "<" ","?            GenericArgumentNamed#   ","? ">"
;

-ParameterGeneric<Optional> ::=
+ParameterGeneric<Named, Optional> ::=
	| (
-		|                    ("_" | IDENTIFIER)
+		| <Named+>(Word "=") ("_" | IDENTIFIER)
+		| <Named+>("$" IDENTIFIER)
	) <Optional+>"?" (("narrows" | "widens") Type)? <Optional+>("=" Type)
;

ParametersGeneric ::=
-	| ","? ParameterGeneric<-Optional># ("," ParameterGeneric<+Optional>#)? ","?
-	| ","?                                   ParameterGeneric<+Optional>#   ","?
+	| ","? ParameterGeneric<-Named><-Optional># ("," ParameterGeneric<-Named><+Optional>#)? ("," ParameterGeneric<+Named><∓Optional>#)? ","?
+	| ","?                                           ParameterGeneric<-Named><+Optional>#   ("," ParameterGeneric<+Named><∓Optional>#)? ","?
+	| ","?                                                                                       ParameterGeneric<+Named><∓Optional>#   ","?
;
```
