Destructuring is a shorthand syntax for declaring multiple variables, properties, parameters, or arguments at a time. With destructuring, we can initialize a static number of identifiers in one line of code, instead of over several lines.

Only tuples and records, which are static, can be destructured. Lists, hashes, maps, and sets cannot be destructured since they are dynamic. Dependent on #21.

There are five areas where destructuring can be used:

- Covered in this issue:

	1. to declare variables

- Not covered in this issue:

	2. to declare properties of a record literal
	3. to reassign variables and properties
	4. to pass named arguments to a function call
	5. to declare function parameters

Additional topics not covered here:
- destructuring defaults for function parameters

# Discussion

## Variable Declaration
Rather than declaring and initializing variables separately …
```cp
let x: int = 42;
let y: int = 420;
```
… we can do so in one concise destructuring statement:
```cp
let [x: int, y: int] = [42, 420];
x; %== 42
y; %== 420
```

Since both `x` and `y` are of type `int`, we can make our declaration even more consise, by moving the type annotation “outside the pattern”. Either of the following lines is sufficient.
```cp
let [x, y]: int[2]     = [42, 420];
let [x, y]: [int, int] = [42, 420];
```

Destructuring applies to unfixed variables as well.
```cp
let [x, unfixed y]: int[2] = [42, 420];
set x = 0; %> AssignmentError
set y = 0; % ok
```

Above, we used **tuple destructuring**, that is, assigning the “pattern” *[`x`, `y`]* a tuple. We can also use **record destructuring** by assigning it a record.
```cp
let [y$: int, x$: int] = [x= 42, y= 420];
y; %== 420
x; %== 42
```
Or, with the “outside” type annotation, if you like:
```cp
let [y$, x$]: [x: int, y: int] = [x= 42, y= 420];
```

As with record punning (#24), the symbol `$` is shorthand for repeating the variable name — `[x$]` is shorthand for `[x= x]`, where the first `x` is the property in the record that we’re destructuring, and the second `x` is the new variable we want to declare. If our record has different property names, we can use aliases.
```cp
let [yankee= y: int, xray= x: int] = [xray= 42, yankee= 420];
y; %== 420
x; %== 42
yankee; %> ReferenceError
xray;   %> ReferenceError
```

Record destructuring has an advantage over tuple destructuring: we can change up the order in which we declare variables. With a tuple, the order of declared variables must match the order of entries in the tuple. With a record, we can switch the order as shown above.

Again, we can assign unfixed variables.
```cp
let [
	w$:              int,
	xray=         x: int,
	y=    unfixed y: int,
	zulu= unfixed z: int,
] = [
	w=       42,
	y=      420,
	xray=  4200,
	zulu= 42000,
];
set w = 0; %> AssignmentError
set x = 0; %> AssignmentError
set y = 0; % ok
set z = 0; % ok
```

## Nested Destructuring
Destructuring for variable declaration can be recursive: We can nest destructured patterns within each other.
```cp
% regular variable destructuring, tuple
let [a, b]: int[2] = [1, 2];

% regular variable destructuring, record
let [c$: int, delta= d: int] = [c= 3, delta= 4];

[a, b, c, d] ==
[1, 2, 3, 4]; %== true

% nested variable destructuring, tuple within tuple
let [g, [h, i]]: [int, int[2]] = [7, [8, 9]];
let [g: int, [h, i]: int[2]]   = [7, [8, 9]];
let [g: int, [h: int, i: int]] = [7, [8, 9]];

% nested variable destructuring, record within tuple
let [j, [k$, lima= l]]: [int, [k: int, lima: int]] = [10, [k= 11, lima= 12]];
let [j: int, [k$, lima= l]: [k: int, lima: int]]   = [10, [k= 11, lima= 12]];
let [j: int, [k$: int, lima= l: int]]              = [10, [k= 11, lima= 12]];

% nested variable destructuring, tuple within record
let [m$, november= [n, o]]: [m: int, november: int[2]] = [m= 13, november= [14, 15]];
let [m$: int, november= [n, o]: int[2]]                = [m= 13, november= [14, 15]];
let [m$: int, november= [n: int, o: int]]              = [m= 13, november= [14, 15]];

% nested variable destructuring, record within record
let [papa= p, quebec= [q$, romeo= r]]: [p: int, quebec: [q: int, romeo: int]] = [papa= 16, quebec= [q= 17, romeo= 18]];
let [papa= p: int, quebec= [q$, romeo= r]: [q: int, romeo: int]]              = [papa= 16, quebec= [q= 17, romeo= 18]];
let [papa= p: int, quebec= [q$: int, romeo= r: int]]                          = [papa= 16, quebec= [q= 17, romeo= 18]];

[g, h, i, j,  k,  l,  m,  n,  o,  p,  q,  r ] ==
[7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18]; %== true
```
Any variable in a nested destructure may be preceded with the modifier `unfixed`.

## Errors and Caveats

### Desugaring
Destructuring is not *perfect* syntax sugar. That is, you could not replace the syntax with its “desugared form” and always expect the same results. In particular, this is due to the fact that the assigned expression is evaluated only once as opposed to multiple times. For example, if the assigned expression is a function call with side-effects, then those side-effects will only be observed once in the destructured form.

Say `f` is a function that generates a pseudo-random number `r` and returns a tuple `[r, r]` of that number duplicated. Then with destructuring two variables, we can be sure the variables are equal.
```cp
let [r1, r2]: float[2] = f.();
r1 == r2; %== true
```
We’re not afforded this certainty if we “desugar” the destructuring statement, since the function is called twice, resulting in two possibly different random numbers.
```cp
let r1: float = f.().0;
let r2: float = f.().1;
r1 == r2; % not necessarily true, in fact most likely false
```

However, destructuring punning with `$` is purely syntactic sugar. For example, `let [x$: int] = [x= 42]` may be replaced with `let [x= x: int] = [x= 42]` at parse-time.

### Type Errors
Type errors are raised when the assigned expression of a destructuring statement doesn’t match the assignee.

Assigning a tuple with greater items is fine, but not fewer items.
```cp
let [a, b, c]: int[3] = [42, 420, 4200, 42000]; % `42000` is dropped, but no error
let [d, e, f]: int[2] = [42, 420];              %> TypeError (index `2` is missing)
```

Assigning a list gives us the same error.
```cp
let list: int[] = [42, 420, 4200, 42000];
let [a, b, c]: int[3] = list;             %> TypeError (index `0` could be missing)
let [d, e, f]: int[] = [42, 420, 4200];   %> TypeError (`int[]` not assignable to `unknown[3]`)
```

Assigning a record with extra properties is fine, but not missing properties.
```cp
let [a$: int, b$: int, c$: int] = [a= 42, b= 420, c= 4200, d= 42000]; % `d` is dropped, but no error
let [d$: int, e$: int, f$: int] = [d= 42, e= 420];                    %> TypeError (property `f` is missing)
let [golf= g: int, hotel= h: int] = [golf= 42, h= 420];               %> TypeError (property `hotel` is missing)
```

Of course, the assigned items must be assignable to the variables’ types.
```cp
let [a, b, c]: int[3]       = [42, 420, 123.45];      %> TypeError (`123.45` is not an int)
let [d$: int, echo= e: int] = [d= null, echo= "420"]; %> TypeError
```

# Specfication
## Syntax Grammar
```diff
+DestructureVariableItem<Typed> ::=
+	| "unfixed"? IDENTIFIER . <Typed+>(":" Type)
+	| DestructureVariables<?Typed>
+	| <Typed+>(DestructureVariables<-Typed> ":" Type)
+;

+DestructureVariableKey<Typed> ::=
+	| IDENTIFIER "$" <Typed+>(":" Type)
+	| Word "=" DestructureVariableItem<?Typed>
+;

+DestructureVariables<Typed> ::=
+	| "[" ","? DestructureVariableItem <?Typed># ","? "]"
+	| "[" ","? DestructureVariableKey  <?Typed># ","? "]"
+;

DeclarationVariable ::=
	| "let" "unfixed"? IDENTIFIER        ":" Type "=" Expression ";"
+	| "let" DestructureVariables<-Typed> ":" Type "=" Expression ";"
+	| "let" DestructureVariables<+Typed>          "=" Expression ";"
;
```
