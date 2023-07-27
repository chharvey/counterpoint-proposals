Destructuring parameters in function definitions.

# Discussion

## Function Parameter Declaration
Consider this function:
```cp
func printPlanet(planet: [str, float]): void {
	'''The radius of {{ planet.0 }} is {{ planet.1 }}km.''';
}
% typeof printPlanet: (planet: [str, float]) => void
```
Rather than keep track of a tuple argument, we can **destructure the tuple** into function parameters.
```cp
func printPlanet(planet as (name: str, value: float)): void {
%                ^ label for destructured parameter
	planet; %> ReferenceError
	'''The radius of {{ name }} is {{ value }}km.''';
}
% typeof printPlanet: (planet: [str, float]) => void
```
Notice the **vector** in the parentheses. The function still has one parameter, a tuple of type `[str, float]`, but inside the function body we can treat each component as if it were a separate parameter. The parentheses distinguish the destructured tuple from a record. Notice also the parameter name `planet`, followed by the keyword `as`. Inside the function body, it’s not defined, but the caller may still use it as the label of a named argument if they wish.

We can make our declaration even more consise by moving the type declaration outside the vector.
```cp
func add(numbers as (a, b): int[2]): void {
	% typeof a == int
	% typeof b == int
	;
}
% typeof add: (numbers: [int, int]) => void
```

Destructuring applies to unfixed parameters as well.
```cp
func add(numbers as (unfixed a, b): int[2]): void {
	set a = a + 1; % ok
	set b = b - 1; %> AssignmentError
}
```

Above, we used **tuple destructuring**, that is, assigning the vector *(`a`, `b`)* a tuple. We can also use **record destructuring** by assigning it a record. Instead of declaring a record parameter…
```cp
func printPlanet(planet: [name: str, value: float]): void {
	'''The radius of {{ planet.name }} is {{ planet.value }}km.''';
}
% typeof printPlanet: (planet: [name: str, value: float]) => void
```
we can destructure the record into separate parameters:
```cp
func printPlanet(planet as (name$: str, value$: float)): void {
	planet; %> ReferenceError
	'''The radius of {{ name }} is {{ value }}km.''';
}
% typeof printPlanet: (planet: [name: str, value: float]) => void
```
As with tuple destructuring, this doesn’t change the type signature of the function. It only allows the function body to reference each component of the vector separately. **Destructuring parameters is an *implementation technique***, which is why type aliases and abstract methods do not allow it.

Recall that with record destructuring for variables (#43), the symbol `$` is shorthand for repeating the variable name — `(x$)` is shorthand for `(x as x)`. This holds for parameters as well. We can replace `$` with internal parameter names.
```cp
func printPlanet(planet as (name as n: str, value as v: float)): void {
	name;  %> ReferenceError
	value; %> ReferenceError
	'''The radius of {{ n }} is {{ v }}km.''';
}
% typeof printPlanet: (planet: [name: str, value: float]) => void
```
The caller must supply a `[name: str, value: float]` argument, but the internal implementation of the function references the parameters `n` and `v`.

Again, we can declare unfixed parameters.
```cp
func f(numbers as (
	w$:                int,
	xray as         x: int,
	y    as unfixed y: int,
	zulu as unfixed z: int,
)): void {
	set w = 0; %> AssignmentError
	set x = 0; %> AssignmentError
	set y = 0; % ok
	set z = 0; % ok
}
```

## Optional Parameters
Optional destructured parameters work as regular parameters; they must be initialized to a value assignable to the correct type.
```cp
func printPlanetNamed(planet as (name: str, value: float) = ['Earth', 6371.0]): void {
	'''The radius of {{ name }} is {{ value }}km.''';
}
% typeof printPlanetNamed: (planet?: [str, float]) => void
```
We can also have optional record destructuring parameters:
```cp
func printPlanetNamed(planet as (name as n: str, value as v: float) = [name= 'Earth', value= 6371.0]): void {
	'''The radius of {{ n }} is {{ v }}km.''';
}
% typeof printPlanetNamed: (planet?: [name: str, value: float]) => void
```

## Nested Destructuring
Like destructuring for variables, we can nest destructuing syntax for functions.
```cp
func nest(
	% regular parameter destructuring, tuple
	ab as (a, b): int[2],

	% regular parameter destructuring, record
	cd as (c$: int, delta as d: int),

	% nested parameter destructuring, tuple within tuple
	ghi as (g: int, (h, i): int[2]),

	% nested parameter destructuring, record within tuple
	jkl as (j: int, (k$: int, lima as l: int)),

	% nested parameter destructuring, tuple within record
	mno as (m$: int, november as (n, o): int[2]),

	% nested parameter destructuring, record within record
	pqr as (papa as p: int, quebec as (q$: int, romeo as r: int)),
): void {
	% typeof each of [a, b, c, d, g, h, i, j, k, l, m, n, o, p, q, r] == int
	uniform;  %> ReferenceError
	delta;    %> ReferenceError
	lima;     %> ReferenceError
	november; %> ReferenceError
	papa;     %> ReferenceError
	quebec;   %> ReferenceError
	romeo;    %> ReferenceError
}
%% typeof nest: (
	ab:  [int, int],
	cd:  [c: int, delta: int],
	ghi: [int, [int, int]],
	jkl: [int, [k: int, lima: int]],
	mno: [m: int, november: [int, int]],
	pqr: [papa: int, quebec: [q: int, romeo: int]],
) => void %%
% example call:
nest.(
	[1, 2],
	[c= 3, delta= 4],
	[7, [8, 9]],
	[10, [k= 11, lima= 12]],
	[m= 13, november= [14, 15]],
	[papa= 16, quebec= [q= 17, romeo= 18]],
);

func nestOptional(
	s as (a, b): int[2]                                         = [19, 20],
	t as (c$: int, delta as d: int)                             = [c= 21, delta= 22],
	u as (g: int, (h, i): int[2])                               = [23, [24, 25]],
	v as (j: int, (k$: int, lima as l: int))                    = [26, [k= 27, lima= 28]],
	w as (m$: int, november as (n, o): int[2])                  = [m= 29, november= [30, 31]],
	x as (papa as p: int, quebec as (q$: int, romeo as r: int)) = [papa= 32, quebec= [q= 33, romeo= 34]],
): void {;}
%% typeof nestOptional: (
	s?: [int, int],
	t?: [c: int, delta: int],
	u?: [int, [int, int]],
	v?: [int, [k: int, lima: int]],
	w?: [m: int, november: [int, int]],
	x?: [papa: int, quebec: [q: int, romeo: int]],
) => void %%
% example call:
nestOptional.(
	s= [1, 2],
	t= [c= 3, delta= 4],
	u= [7, [8, 9]],
	v= [10, [k= 11, lima= 12]],
	w= [m= 13, november= [14, 15]],
	x= [papa= 16, quebec= [q= 17, romeo= 18]],
);
```

# Specfication

## Syntax Grammar
```diff
ParameterFunction<Optional> ::=
		| (IDENTIFIER "as")? "unfixed"? IDENTIFIER        ":" Type  . <Optional+>("=" Expression)
+		|  IDENTIFIER "as"   DestructureVariables<-Typed> ":" Type  . <Optional+>("=" Expression)
+		|  IDENTIFIER "as"   DestructureVariables<+Typed>           . <Optional+>("=" Expression)
;
```
