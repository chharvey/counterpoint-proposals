Like function parameters (#46), type parameters (of a type alias or type function) may be named. The name may be supplied when **specified** (“called”).

```cp
type Union<Left= A, Right= B> = A | B;
type Intersection<Left= A, Right= B ?= null> = A & B;

type IntOrFloat = Union.<Left=  int,   Right= float>; %== int   | float
type IntOrFloat = Union.<Right= float, Left=  int>;   %== int   | float
type FloatOrInt = Union.<float,        Right= int>;   %== float | int
type FloatOrInt = Union.<Left= float,  int>;          %> ParseError

type IntAndNull = Intersection.<Left= int>;  %== int & null
type IntAndNull = Intersection.<int>;        %== int & null
type IntAndNull = Intersection.<Right= int>; %> TypeError % expected an argument for parameter `Left`
```
Identifiers `A` and `B` are accessible at the “define site”; identifiers `Left` and `Right` are accessible at the “call site”.

The same rules of named arguments (#57) apply:

- All positional arguments must come before all named arguments.
- All positional arguments are assigned first, followed by all named arguments.
- All argument values are evaluated in source order.
- After all given arguments are assigned, any unspecified optional parameters receive their default value (#66).

## Syntax
```diff
ParameterGeneric<Optional>
-	::=                   IDENTIFIER (("narrows" | "widens") Type)? <Optional+>("?=" Type);
+	::= (IDENTIFIER "=")? IDENTIFIER (("narrows" | "widens") Type)? <Optional+>("?=" Type);
```
