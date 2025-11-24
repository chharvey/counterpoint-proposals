Destructuring parameters in function definitions.

# Discussion

## Function Parameter Declaration
Consider this function:
```cpl
function printPlanet(planet: (str, float)): void {
	"""The radius of {{ planet.0 }} is {{ planet.1 }}km.""";
}
% typeof printPlanet: \((str, float)) => void
```
Rather than keep track of a tuple argument, we can **destructure the tuple** into function parameters.
```cpl
function printPlanet((name, value): (str, float)): void {
%                    ^ destructured parameter
	"""The radius of {{ name }} is {{ value }}km.""";
}
% typeof printPlanet: \((str, float)) => void
```
Notice the “pattern” in the brackets. The function still has one parameter, a tuple of type `(str, float)`, but inside the function body we can treat each component as if it were a separate parameter.

We can make our declaration a litle more consise by moving the typings inside the destructure pattern.
```cpl
function printPlanet((name: str, value: float)): void {
	% typeof name  == str
	% typeof value == float
	;
}
```

Destructuring applies to unfixed parameters as well.
```cpl
function printPlanet((var name, value): (str, float)): void {
	set name  = """Planet {{ name }}"""; % ok
	set value = value + 1.0;             %> AssignmentError
}
```

The syntax for named parameters (#46) is the same as before; just prepend the destructured pattern with the name of the parameter and an equals sign `=`. When the parameter is named, the caller is expected to provide that label for the argument when calling the function.
```cpl
function printPlanet(planet= (name, value): (str, float)): void {
%                    ^ label for destructured parameter
	planet; %> ReferenceError
	"""The radius of {{ name }} is {{ value }}km.""";
}
% typeof printPlanet: \(planet: (str, float)) => void
printPlanet.(planet= my_planet);
```
Because the purpose of destructuring is to simplify parameter names inside the function body, named destructured parameters cannot be “punned” (using the `$` sigil).
```cpl
function printPlanet($(name, value): (str, float)): void {;} %> SyntaxError
```

Above, we used **tuple destructuring**, that is, assigning the pattern *(`a`, `b`)* a tuple. We can also use **record destructuring** by assigning it a record. Instead of declaring a record parameter…
```cpl
function printPlanet((name: str, value: float)): void {
	"""The radius of {{ planet.name }} is {{ planet.value }}km.""";
}
% typeof printPlanet: \((name: str, value: float)) => void
```
we can destructure the record into separate parameters:
```cpl
function printPlanet(($name, $value): (name: str, value: float)): void {
	"""The radius of {{ name }} is {{ value }}km.""";
}
```
Or, with the “inside” type annotation, if you like:
```cpl
function printPlanet(($name: str, $value: float)): void {
	"""The radius of {{ name }} is {{ value }}km.""";
}
```
The “record punning” (`$`) is explained below.

The parameter can also be named:
```cpl
function printPlanet(planet= ($name: str, $value: float)): void {
	planet; %> ReferenceError
	"""The radius of {{ name }} is {{ value }}km.""";
}
% typeof printPlanet: \(planet: (name: str, value: float)) => void
printPlanet.(planet= my_planet);
```

As with tuple destructuring, this doesn’t change the type signature of the function. It only allows the function body to reference each component of the pattern separately. **Destructuring parameters is an *implementation technique***, which is why abstract methods don’t allow it.

Recall that with record destructuring for variables (#43), the symbol `$` is shorthand for repeating the variable name — `($x)` is shorthand for `(x= x)`. This is called “punning”. This holds for parameters as well. We can replace `$` with an internal parameter names.
```cpl
function printPlanet((name= n: str, value= v: float)): void {
	name;  %> ReferenceError
	value; %> ReferenceError
	"""The radius of {{ n }} is {{ v }}km.""";
}
% typeof printPlanet: \((name: str, value: float)) => void
```
The caller must supply a `(name: str, value: float)` argument, but the internal implementation of the function references the parameters `n` and `v`.

Again, we can declare unfixed parameters.
```cpl
function f((
	$w:          int, % punning for `w= w: int`
	xray=     x: int,
	var $y:      int, % punning for `y= var y: int`
	zulu= var z: int,
)): void {
	set w = 0; %> AssignmentError
	set x = 0; %> AssignmentError
	set y = 0; % ok
	set z = 0; % ok
}
```

## Optional Parameters
Optional destructured parameters work just like regular parameters; they must be initialized to a value assignable to the correct type.
```cpl
function printPlanetNamed((name: str, value: float) ?= ("Earth", 6371.0)): void {
	"""The radius of {{ name }} is {{ value }}km.""";
}
% typeof printPlanetNamed: \(?: (str, float)) => void
```
We can also have optional record destructuring parameters:
```cpl
function printPlanetNamed((name= n: str, value= v: float) ?= (name= "Earth", value= 6371.0)): void {
	"""The radius of {{ n }} is {{ v }}km.""";
}
% typeof printPlanetNamed: \(?: (name: str, value: float)) => void
```

## Nested Destructuring
Like destructuring for variables, we can nest destructuing syntax for functions.
```cpl
function nest(
	% regular parameter destructuring, tuple
	ab= (a, b): (int, int),

	% regular parameter destructuring, record
	cd= ($c: int, delta= d: int),

	% nested parameter destructuring, tuple within tuple
	ghi= (g: int, (h, i): (int, int)),

	% nested parameter destructuring, record within tuple
	jkl= (j: int, ($k: int, lima= l: int)),

	% nested parameter destructuring, tuple within record
	mno= ($m: int, november= (n, o): (int, int)),

	% nested parameter destructuring, record within record
	pqr= (papa= p: int, quebec= ($q: int, romeo= r: int)),
): void {
	% typeof each of (a, b, c, d, g, h, i, j, k, l, m, n, o, p, q, r) == int
	uniform;  %> ReferenceError
	delta;    %> ReferenceError
	lima;     %> ReferenceError
	november; %> ReferenceError
	papa;     %> ReferenceError
	quebec;   %> ReferenceError
	romeo;    %> ReferenceError
}
%% typeof nest: \(
	ab:  (int, int),
	cd:  (c: int, delta: int),
	ghi: (int, (int, int)),
	jkl: (int, (k: int, lima: int)),
	mno: (m: int, november: (int, int)),
	pqr: (papa: int, quebec: (q: int, romeo: int)),
) => void %%
% example call:
nest.(
	ab=  (1, 2),
	cd=  (c= 3, delta= 4),
	ghi= (7, (8, 9)),
	jkl= (10, (k= 11, lima= 12)),
	mno= (m= 13, november= (14, 15)),
	pqr= (papa= 16, quebec= (q= 17, romeo= 18)),
);

function nestOptional(
	s= (a, b): (int, int)                               ?= (19, 20),
	t= ($c: int, delta= d: int)                         ?= (c= 21, delta= 22),
	u= (g: int, (h, i): (int, int))                     ?= (23, (24, 25)),
	v= (j: int, ($k: int, lima= l: int))                ?= (26, (k= 27, lima= 28)),
	w= ($m: int, november= (n, o): (int, int))          ?= (m= 29, november= (30, 31)),
	x= (papa= p: int, quebec= ($q: int, romeo= r: int)) ?= (papa= 32, quebec= (q= 33, romeo= 34)),
): void {;}
%% typeof nestOptional: \(
	s?: (int, int),
	t?: (c: int, delta: int),
	u?: (int, (int, int)),
	v?: (int, (k: int, lima: int)),
	w?: (m: int, november: (int, int)),
	x?: (papa: int, quebec: (q: int, romeo: int)),
) => void %%
% example call:
nestOptional.(
	s= (1, 2),
	t= (c= 3, delta= 4),
	u= (7, (8, 9)),
	v= (10, (k= 11, lima= 12)),
	w= (m= 13, november= (14, 15)),
	x= (papa= 16, quebec= (q= 17, romeo= 18)),
);
```

# Specfication

## Syntax Grammar
```diff
ParameterFunction<Optional> ::=
		| (Word "=")? "var"? ("_" | IDENTIFIER)    ":" Type  & <Optional+>("?=" Expression)
+		|  Word "="   DestructureVariables<-Typed> ":" Type  & <Optional+>("?=" Expression)
+		|  Word "="   DestructureVariables<+Typed>           & <Optional+>("?=" Expression)
;
```
