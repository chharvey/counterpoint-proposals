Dependent on #15, #24, and #57.

```
func myFunction(
	a: int,
	b: int = default_b,
	c: int = default_c,
	d: int = default_d,
	e: int = default_e,
): void => ();

let b: int = 42;
let c: int = 420;
let d: int = 4200;
let e: int = 42000;
let f: int = 24;
myFunction(
	42,
	b += f, % b= b + f
	c++,    % c += 1
	d= $,   % d= d
	e += $, % e += e;
)~~;
```

```diff
Label
-	::= IDENTIFIER   "="                                           Expression;
+	::= IDENTIFIER (("=" | AugmentOperator | AugmentNegate) ("$" | Expression) | UpdateOperator);
```
