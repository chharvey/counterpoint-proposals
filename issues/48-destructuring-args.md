Destructuring named arguments for function calls.

When we want to send named parameters into a function call, we may destructure the argument labels.
Notice the function specifies named parameters, which means arguments must be named when called.
```cpl
func pythag($side1: float, $side2: float, $hypotenuse: float): bool
	=> side1 ^ 2 + side2 ^ 2 == hypotenuse ^ 2;
```
We can call this function in three ways:
```cpl
% no destructuring:
pythag(side1= 3.0, side2= 4.0, hypotenuse= 5.0);

% positional destructuring:
val tuple: (float, float, float) = (3.0, 4.0, 5.0);
pythag((side1, side2, hypotenuse)= tuple);
% shorthand for:
% pythag(side1= tuple.0, side2= tuple.1, hypotenuse= tuple.2);

% named destructuring:
val record: (a: float, b: float, c: float) = (a= 3.0, b= 4.0, c= 5.0);
pythag((a= side1, b= side2, c= hypotenuse)= record);
% shorthand for:
% pythag(side1= record.a, side2= record.b, hypotenuse= record.c);
```

The arguments do not necessarily need to be destructured together; we can split them up and rearrange them:
```cpl
% positional destructuring:
val tuple: (float, float, float) = (3.0, 4.0, 5.0);
pythag(
	(hypotenuse)=   (5.0),
	(side1, side2)= tuple,
);
% shorthand for:
% pythag(hypotenuse= 5.0, side1= tuple.0, side2= tuple.1);

% named destructuring:
val record: (a: float, b: float, c: float) = (a= 3.0, b= 4.0, c= 5.0);
pythag(
	(c= hypotenuse)=      (c= 5.0),
	(b= side2, a= side1)= record,
);
% shorthand for:
% pythag(hypotenuse= 5.0, side2= record.b, side1= record.a);
```

Function argument destructuring can be nested as well.
```cpl
func nest(
	a: int, b: int, c: int, d: int, g: int, h: int, i: int, j: int,
	k: int, l: int, m: int, n: int, o: int, p: int, q: int, r: int,
): void {}
nest(
	% regular argument destructuring, positional
	(a, b)= (1, 2),

	% regular argument destructuring, named
	($c, delta= d)= (c= 3, delta= 4),

	% nested argument destructuring, positional within positional
	(g, (h, i))= (7, (8, 9)),

	% nested argument destructuring, named within positional
	(j, ($k, lima= l))= (10, (k= 11, lima= 12)),

	% nested argument destructuring, positional within named
	($m, november= (n, o))= (m= 13, november= (14, 15)),

	% nested argument destructuring, named within named
	(papa= p, quebec= ($q, romeo= r))= (papa= 16, quebec= (q= 17, romeo= 18)),
);
% shorthand for:
%% nest(
	a=  1, b=  2, c=  3, d=  4, g=  7, h=  8, i=  9, j= 10,
	k= 11, l= 12, m= 13, n= 14, o= 15, p= 16, q= 17, r= 18,
); %%
```

# Syntax
```diff
DestructureLabel<Named, Colon> ::=
	| <Named+>(Word & <Colon->"=" <Colon+>":" | "$") Word
	| <Named+>(Word & <Colon->"=" <Colon+>":")       DestructureLabels
;

DestructureLabels<Colon>
	::= "(" ","? (DestructureLabel<-Named><?Colon># | DestructureLabel<+Named><?Colon>#) ","? ")";


GenericArgumentNamed ::=
	| "$" IDENTIFIER
-	|  Word                              "=" Type
+	| (Word | DestructureLabels<-Colon>) "=" Type
;

GenericArguments ::=
	| "<" ","? Type# ("," GenericArgumentNamed#)? ","? ">"
	| "<" ","?            GenericArgumentNamed#   ","? ">"
;
```
