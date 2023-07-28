Destructuring for variable & property reassignment. Depends on #43.

# Discussion
Destructuring reassignment is shorthand syntax for reassignment of variables and properties. As with variable and property declaration, we can use either tuple or record destructuring.
```cp
let unfixed x: int = 42;
let unfixed y: int = 420;

% tuple destructuring:
set [x, y] = [43, 430];
x == 43;
y == 430;

% record destructuring:
set [yankee= y, xray= x] = [xray= 44, yankee= 440];
x == 44;
y == 440;

% record destructuring, punning:
set [y$, x$] = [x= 45, y= 450];
x == 45;
y == 450;
```

Destructuring for index/key reassignment allows us to reassign items on mutable tuples and properties on mutable records. Note: Punning (`k$`) can only be used for variable reassignment, not for property reassignment.
```cp
let t: mutable [float, float] = [4.2, 0.42];
let r: mutable [x: int, y: int] = [x= 42, y= 420];

set [t.0, t.1] = [4.3, 0.43];
set [r.x, r.y] = [43, 430];
set [yankee= r.y, xray= r.x] = [xray= 44, yankee= 440];
set [y$, x$] = [x= 45, y= 450];                         % ReferenceError: `x` and `y` not defined
```

Destructuring for reassignment can be nested as well.
```cp
% nested reassignment, tuple within tuple
set [object.g, (object.h, object.i)] = [7, [8, 9]];

% nested reassignment, record within tuple
set [object.j, (k$, lima= object.l)] = [10, [k= 11, lima= 12]];

% nested reassignment, tuple within record
set [m$, november= (object.n, object.o)] = [m= 13, november= [14, 15]];

% nested reassignment, record within record
set [papa= object.p, quebec= (q$, romeo= object.r)] = [papa= 16, quebec= [q= 17, romeo= 18]];

[object.g, object.h, object.i, object.j, k,  object.l, m,  object.n, object.o, object.p, q,  object.r ] ==
[7,        8,        9,        10,       11, 12,       13, 14,       15,       16,       17, 18       ];
```

With destructuring for reassignment, we can use the variables’ previous values.
```cp
let unfixed x: int = 42;
let unfixed y: int = 420;

set [x, y] = [x + 1, y + 10];
x == 43;
y == 430;

set [x, y] = [y, x];
x == 430;
y == 43;
```

We can reassign variables in the same destructuring statement without affecting each other.
```cp
let unfixed x: int = 42;
let unfixed y: int = 0;

set [x, y] = [x + 2, x * 10]; % evaluated first as `[44, 420]`
x == 44;
y == 420; % not 440
```
`y` is `420` instead of `440` because the destructured tuple is evaluated as `[44, 420]` before any assignments are made.

# Specification
```diff
Assignee ::=
	| IDENTIFIER
	| ExpressionCompound PropertyAssign
;

DestructurePropertyItem  ::= Word     | DestructureProperties;
+DestructureAssigneeItem ::= Assignee | DestructureAssignees;

DestructurePropertyKey  ::= Word       "$" | Word "=" DestructurePropertyItem;
+DestructureAssigneeKey ::= IDENTIFIER "$" | Word "=" DestructureAssigneeItem;

DestructureProperties ::=
	| "[" ","? DestructurePropertyItem# ","? "]"
	| "[" ","? DestructurePropertyKey#  ","? "]"
;

+DestructureAssignees ::=
+	| "[" ","? DestructureAssigneeItem# ","? "]"
+	| "[" ","? DestructureAssigneeKey#  ","? "]"
+;

DeclarationReassignment ::=
	| "set" Assignee             ("=" | AugmentOperator | AugmentNegate) Expression ";"
+	| "set" DestructureAssignees  "="                                    Expression ";"
;
