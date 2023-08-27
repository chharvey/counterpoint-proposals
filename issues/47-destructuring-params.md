Destructuring parameters in function definitions.

# Discussion

## Function Parameter Declaration
Consider this function:
```cp
func printPlanet(planet: [str, float]): void {
	"""The radius of {{ planet.0 }} is {{ planet.1 }}km.""";
}
% typeof printPlanet: (planet: [str, float]) => void
```
Rather than keep track of a tuple argument, we can **destructure the tuple** into function parameters.
```cp
func printPlanet(planet= [name, value]: [str, float]): void {
%                ^ label for destructured parameter
	planet; %> ReferenceError
	"""The radius of {{ name }} is {{ value }}km.""";
}
% typeof printPlanet: (planet: [str, float]) => void
```
Notice the “pattern” in the brackets. The function still has one parameter, a tuple of type `[str, float]`, but inside the function body we can treat each component as if it were a separate parameter. Notice also the parameter name `planet`, followed by the symbol `=` used for aliasing (#46). It’s not defined inside the function body, but the caller may still use it as the label of a named argument if they wish.

We can make our declaration a litle more consise by moving the typings inside the destructure pattern.
```cp
func printPlanet(planet= [name: str, value: float]): void {
	% typeof name  == str
	% typeof value == float
	;
}
```

Destructuring applies to unfixed parameters as well.
```cp
func printPlanet(planet= [unfixed name, value]: [str, float]): void {
	set name  = """Planet {{ name }}"""; % ok
	set value = value + 1.0;             %> AssignmentError
}
```

Above, we used **tuple destructuring**, that is, assigning the pattern *[`a`, `b`]* a tuple. We can also use **record destructuring** by assigning it a record. Instead of declaring a record parameter…
```cp
func printPlanet(planet: [name: str, value: float]): void {
	"""The radius of {{ planet.name }} is {{ planet.value }}km.""";
}
% typeof printPlanet: (planet: [name: str, value: float]) => void
```
we can destructure the record into separate parameters:
```cp
func printPlanet(planet= [name$, value$]: [name: str, value: float]): void {
	planet; %> ReferenceError
	"""The radius of {{ name }} is {{ value }}km.""";
}
```
Or, with the “inside” type annotation, if you like:
```cp
func printPlanet(planet= [name$: str, value$: float]): void {
	"""The radius of {{ name }} is {{ value }}km.""";
}
```

As with tuple destructuring, this doesn’t change the type signature of the function. It only allows the function body to reference each component of the pattern separately. **Destructuring parameters is an *implementation technique***, which is why type aliases and abstract methods do not allow it.

Recall that with record destructuring for variables (#43), the symbol `$` is shorthand for repeating the variable name — `[x$]` is shorthand for `[x= x]`. This is called “punning”. This holds for parameters as well. We can replace `$` with internal parameter names.
```cp
func printPlanet(planet= [name= n: str, value= v: float]): void {
	name;  %> ReferenceError
	value; %> ReferenceError
	"""The radius of {{ n }} is {{ v }}km.""";
}
% typeof printPlanet: (planet: [name: str, value: float]) => void
```
The caller must supply a `[name: str, value: float]` argument, but the internal implementation of the function references the parameters `n` and `v`.

Again, we can declare unfixed parameters.
```cp
func f(numbers= [
	w$:              int,
	xray=         x: int,
	y=    unfixed y: int,
	zulu= unfixed z: int,
]): void {
	set w = 0; %> AssignmentError
	set x = 0; %> AssignmentError
	set y = 0; % ok
	set z = 0; % ok
}
```

## Optional Parameters
Optional destructured parameters work just like regular parameters; they must be initialized to a value assignable to the correct type.
```cp
func printPlanetNamed(planet= [name: str, value: float] ?= ["Earth", 6371.0]): void {
	"""The radius of {{ name }} is {{ value }}km.""";
}
% typeof printPlanetNamed: (planet?: [str, float]) => void
```
We can also have optional record destructuring parameters:
```cp
func printPlanetNamed(planet= [name= n: str, value= v: float] ?= [name= "Earth", value= 6371.0]): void {
	"""The radius of {{ n }} is {{ v }}km.""";
}
% typeof printPlanetNamed: (planet?: [name: str, value: float]) => void
```

## Nested Destructuring
Like destructuring for variables, we can nest destructuing syntax for functions.
```cp
func nest(
	% regular parameter destructuring, tuple
	ab= [a, b]: int[2],

	% regular parameter destructuring, record
	cd= [c$: int, delta= d: int],

	% nested parameter destructuring, tuple within tuple
	ghi= [g: int, [h, i]: int[2]],

	% nested parameter destructuring, record within tuple
	jkl= [j: int, [k$: int, lima= l: int]],

	% nested parameter destructuring, tuple within record
	mno= [m$: int, november= [n, o]: int[2]],

	% nested parameter destructuring, record within record
	pqr= [papa= p: int, quebec= [q$: int, romeo= r: int]],
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
	s= [a, b]: int[2]                                   ?= [19, 20],
	t= [c$: int, delta= d: int]                         ?= [c= 21, delta= 22],
	u= [g: int, [h, i]: int[2]]                         ?= [23, [24, 25]],
	v= [j: int, [k$: int, lima= l: int]]                ?= [26, [k= 27, lima= 28]],
	w= [m$: int, november= (n, o): int[2]]              ?= [m= 29, november= [30, 31]],
	x= [papa= p: int, quebec= [q$: int, romeo= r: int]] ?= [papa= 32, quebec= [q= 33, romeo= 34]],
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
		| (IDENTIFIER "=")? "unfixed"? IDENTIFIER        ":" Type  . <Optional+>("?=" Expression)
+		|  IDENTIFIER "="   DestructureVariables<-Typed> ":" Type  . <Optional+>("?=" Expression)
+		|  IDENTIFIER "="   DestructureVariables<+Typed>           . <Optional+>("?=" Expression)
;
```
