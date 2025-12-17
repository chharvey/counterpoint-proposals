Destructuring for variable & property reassignment. Depends on #43.

# Discussion
Destructuring reassignment is shorthand syntax for reassignment of variables and properties. As with variable and property declaration, we can use either tuple or record destructuring.
```cpl
let var x: int = 42;
let var y: int = 420;

% tuple destructuring:
set (x, y) = (43, 430);
x == 43;
y == 430;

% record destructuring:
set (yankee= y, xray= x) = (xray= 44, yankee= 440);
x == 44;
y == 440;

% record destructuring, punning:
set ($y, $x) = (x= 45, y= 450);
x == 45;
y == 450;
```

Destructuring for index/key reassignment allows us to reassign entries on mutable lists and dicts. Note: Punning (`k$`) can only be used for variable reassignment, not for property reassignment.
```cpl
let list: mut [float] = [4.2, 0.42];
let dict: mut [:int]  = [x= 42, y= 420];

set (list.[0], list.[1]) = (4.3, 0.43);
set (dict.[@x], dict.[@y]) = (43, 430);
set (yankee= list.[0], xray= list.[1]) = (xray= 4.3, yankee= 0.43);
set (yankee= dict.[@y], xray= dict.[@x]) = (xray= 44, yankee= 440);
set ($y, $x) = (x= 45, y= 450); % ReferenceError: `x` and `y` not defined
```

Destructuring for reassignment can be nested as well.
```cpl
% nested reassignment, tuple within tuple
set (g, (h, i)) = (7, (8, 9));

% nested reassignment, record within tuple
set (j, ($k, lima= l)) = (10, (k= 11, lima= 12));

% nested reassignment, tuple within record
set ($m, november= (n, o)) = (m= 13, november= (14, 15));

% nested reassignment, record within record
set ($p, quebec= ($q, romeo= r)) = (p= 16, quebec= (q= 17, romeo= 18));

(g, h, i,  j,  k,  l,  m,  n,  o,  p,  q,  r) ==
(7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18); %== true
```

With destructuring for reassignment, we can use the variables’ previous values.
```cpl
let var x: int = 42;
let var y: int = 420;

set (x, y) = (x + 1, y + 10);
x == 43;
y == 430;

set (x, y) = (y, x);
x == 430;
y == 43;
```

We can reassign variables in the same destructuring statement without affecting each other.
```cpl
let var x: int = 42;
let var y: int = 0;

set (x, y) = (x + 2, x * 10); % evaluated first as `(44, 420)`
x == 44;
y == 420; % not 440
```
`y` is `420` instead of `440` because the destructured tuple is evaluated as `(44, 420)` before any assignments are made.

# Specification
```diff
Assignee<Break> ::=
	| IDENTIFIER
	| ExpressionCompound<+Block><?Break> "." PropertyAccessor<?Break>
;

DestructureProperty<Named> ::=
	| <Named+>(Word "=" | "$") Word
	| <Named+>(Word "=")       DestructureProperties
;

+DestructureAssignee<Named> ::=
+	| <Named+>"$"        IDENTIFIER
+	| <Named+>(Word "=") (Assignee | DestructureAssignees)
+;

DestructureProperties
	::= "(" ","? (DestructureProperty<-Named># | DestructureProperty<+Named>#) ","? ")";

+DestructureAssignees
+	::= "(" ","? (DestructureAssignee<-Named># | DestructureAssignee<+Named>#) ","? ")";

DeclarationReassignment ::=
	| "set" Assignee             ("=" | AugmentOperator | AugmentNegate) Expression ";"
+	| "set" DestructureAssignees  "="                                    Expression ";"
;
