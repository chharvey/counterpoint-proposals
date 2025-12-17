Default values for destructuring variables and parameters. Depends on #43, #47, #71, #55.

# Discussion
When declaring variables or parameters via a destructuring pattern, default values may be provided in the pattern for cases when a variable/parameter is not assigned.

## Variable Destructuring
When destructuring a tuple, the number of entries in the assigned object must match at least the number of assignee variables. Here we’re attempting to assign 3 variables to a destructured tuple of only 2 entries. This restriction also applies to record keys.
```cpl
let (a, b, c): (int, int, int) = (1, 2); %> TypeError: (index `2` is missing)
let (a, b, c): (int, int) = (1, 2); %> AssignmentError: `c` is not initialized

let ($d: int, $e: int, $f: int)   = (d= 42, e= 420);    %> TypeError (property `f` is missing)
let (golf= g: int, hotel= h: int) = (golf= 42, h= 420); %> TypeError (property `hotel` is missing)
```

These can be lifted by providing **default values**.
```cpl
let (a: int, b: int, c: int ?= 3) = (1, 2);
a; %== 1
b; %== 2
c; %== 3
```
The default value of a destructured variable follows the `?=` symbol, just like the default value of a function parameter. Notice here we’ve moved the types to inside the destructure pattern. In tuple destructuring, all optional entries must come after all required entries.

For record destructuring, default values don’t need to come last (unlike optional function parameters) — a required entry may follow an optional one.
```cpl
let (a= alfa: int, b= bravo: int ?= 3, c= charlie: int) = (a= 1, c= 2);
alfa;    %== 1
bravo;   %== 3
charlie; %== 2
```
(For clarity, and per standard coding style, the whitespace between record key and `=` is omitted.)

As usual, we can use `$` punning to assign variables with the same name as the record key.
```cpl
let ($a: int, $b: int ?= 3, $c: int) = (a= 1, c= 2);
a; %== 1
b; %== 3
c; %== 2
```

## Parameter Destructuring
Function parameters are destructured the same way.
```cpl
function f(arg= (a: int, b: int, c: int ?= 3)): anything {
	arg; %> ReferenceError
	return (a, b, c);
}
% typeof f: \(arg: (int, int, ?: int)) => anything
f.((1, 2)); %== (1, 2, 3)

function g(arg= (a= alfa: int, b= bravo: int ?= 3, c= charlie: int)): anything {
	arg; %> ReferenceError
	a;   %> ReferenceError
	b;   %> ReferenceError
	c;   %> ReferenceError
	return (alfa, bravo, charlie);
}
% typeof g: \(arg: (a: int, b?: int, c: int)) => anything
g.((a= 1, c= 2)); %== (1, 3, 2)

function h(arg= ($a: int, $b: int ?= 3, $c: int)): anything {
	arg; %> ReferenceError
	return (a, b, c);
};
% typeof h: \(arg: (a: int, b?: int, c: int)) => anything
h.((a= 1, c= 2)); %== (1, 3, 2)
```
Since named function parameters are required, we must **alias** the destructured parameter with the `=` symbol (#46). In the examples above, callers may assign the provided parameter to the named argument `arg`, even though that identifier isn’t available in the body.

Just because a destructured function parameter may have optional entries doesn’t mean the *entire parameter* is optional. The three examples above all have required parameters (an argument is required when calling). If we want the parameter to be optional, we must provide a *separate* default value (#55).
```cpl
function f(arg= (a: int, b: int, c: int ?= 3) ?= (4, 5, 6)): anything => (a, b, c);
% typeof f: \(arg?: (int, int, ?: int)) => anything
f.((1, 2)); %== (1, 2, 3)
f.();       %== (4, 5, 6)

function h(arg= ($a: int, $b: int ?= 3, $c: int) ?= (a= 4, c= 5)): anything => (a, b, c);
% typeof h: \(arg?: (a: int, b?: int, c: int)) => anything
h.((a= 1, c= 2)); %== (1, 3, 2)
h.();             %== (4, 3, 5)
```

## Syntax Note
- Tuple destructuring, declaring variables `a`, `b`, `c` with respective default values `x`, `y`, `z`:
	```cpl
	let (a ?= x, b ?= y, c ?= z): MyTupleType = my_tuple;
	a; % set to `my_tuple.0` or `x`
	b; % set to `my_tuple.1` or `y`
	c; % set to `my_tuple.2` or `z`
	```
- Record destructuring, declaring variables `x`, `y`, `z` with no default values:
	```cpl
	let (a= x, b= y, c= z): MyRecordType = my_record;
	x; % set to `my_record.a`
	y; % set to `my_record.b`
	z; % set to `my_record.c`
	```

## Nested Destructuring
Nested destructuring works the same as before. With nesting, the type annotation doesn’t need to go on the entry itself, as long as its innards are all typed.
```cpl
let (a: int, b: int, (c, d): int(2)        ?= (3, 4)) = (1, 2); % the typing can go on the optional entry,
let (a: int, b: int, (c: int, d: int)      ?= (3, 4)) = (1, 2); % or it can go in each of the nested entries.
let (a: int, b: int, (c: int, d: int ?= 5) ?= (3, 4)) = (1, 2); % we can even have nested defaults!

let (a: int, b: int, (c, d ?= 6): (int, ?: int) ?= (4, 5)) = (1, 2, (3));
let (a: int, b: int, (c: int, d: int ?= 6)      ?= (4, 5)) = (1, 2, (3));

let (a: int, b: int, (c, (d) ?= (6)): (int, ?: (int)) ?= (4, (5))) = (1, 2, (3));
let (a: int, b: int, (c: int, (d): (int)    ?= (6))   ?= (4, (5))) = (1, 2, (3));
let (a: int, b: int, (c: int, (d: int)      ?= (6))   ?= (4, (5))) = (1, 2, (3));
let (a: int, b: int, (c: int, (d: int ?= 7) ?= (6))   ?= (4, (5))) = (1, 2, (3, ())); % we can go deeper…
```

## Execution Order
As with function parameter defaults, the default value is only evaluated if being assigned. If an assigned value is provided, the default value is ignored.
```cpl
let ($a: int, $b: int, $c: int ?= some_fn_with_side_effects.()) = (a= 1, c= 2, b= 3);
```
Since `c` is assigned `2`, the function call is not executed and its side effects are not observed.

If *multiple* default values are evaluated, they are done so left-to-right in source code order.
```cpl
let ($a: int, $b: int ?= print_and_return_2.(), $c: int ?= print_and_return_3.()) = (a= 1);
```
After assigning `a`, the program will print `2` and assign `b`, then print `3` and assign `c`.

However, as demonstrated in #65, it’s important to remember that the assigned object is *completely evaluated* before any assignments take place. This means that any function calls in the assigned object are evaluated in source order, regardless of variable assignment order.
```cpl
let ($a: int, $b: int, $c: int) = (c= print_and_return_3.(), b= print_and_return_2.(), a= 1);
```
The program first prints `3` and `2` in that order, then assigns variables `a`, `b`, and `c` in that order.


# Specification

## Syntax
```diff
-DestructureVariable<Named,                          Typed> ::=
+DestructureVariable<Named, Optional, Break, Return, Typed> ::=
-	|          <Named->"var"? <Named+>(Word "=" "var"? | "var"? "$") ("_" | IDENTIFIER) <Typed+>(":" Type)
-	|                         <Named+>(Word "=")                     DestructureVariables<?Typed>
-	| <Typed+>(               <Named+>(Word "=")                     DestructureVariables<-Typed> ":" Type)
+	|          <Named->"var"? <Named+>(Word "=" "var"? | "var"? "$") ("_" | IDENTIFIER) <Typed+>(":" Type)   <Optional+>("?=" Expression<+Block><?Break><?Return>)
+	|                         <Named+>(Word "=")                     DestructureVariables<?Typed>          & <Optional+>("?=" Expression<+Block><?Break><?Return>)
+	| <Typed+>(               <Named+>(Word "=")                     DestructureVariables<-Typed> ":" Type & <Optional+>("?=" Expression<+Block><?Break><?Return>))
;

-DestructureVariables<Typed>                ::= "(" ","? (
+DestructureVariables<Break, Return, Typed> ::= "(" ","? (
-	| DestructureVariable<-Named><?Typed>#
-	| DestructureVariable<+Named><?Typed>#
+	| DestructureVariable<-Named><-Optional><?Break><?Return><?Typed># ("," DestructureVariable<-Named><+Optional><?Break><?Return><?Typed>#)?
+	|                                                                       DestructureVariable<-Named><+Optional><?Break><?Return><?Typed>#
+	| DestructureVariable<+Named><∓Optional><?Break><?Return><?Typed>#
) ","? ")";

DeclarationVariable<Break, Return> ::=
	| "let" "var"  ("_" | IDENTIFIER)    "?:" Type                                         ";"
	| "let" "var"? ("_" | IDENTIFIER)    ":"  Type "=" Expression<+Block><?Break><?Return> ";"
-	| "let" DestructureVariables                 <-Typed> ":"  Type "=" Expression<+Block><?Break><?Return> ";"
-	| "let" DestructureVariables                 <+Typed>           "=" Expression<+Block><?Break><?Return> ";"
+	| "let" DestructureVariables<?Break><?Return><-Typed> ":"  Type "=" Expression<+Block><?Break><?Return> ";"
+	| "let" DestructureVariables<?Break><?Return><+Typed>           "=" Expression<+Block><?Break><?Return> ";"
;

ParameterFunction<Named, Optional> ::=
	| <Named->"var"? <Named+>(Word "=" "var"? | "var"? "$") ("_" | IDENTIFIER)                            <Optional->(":" Type)   <Optional+>("?:" Type | ":" Type "?=" Expression<+Block><-Break><-Return>)
-	|                <Named+>(Word "=")                     DestructureVariables                 <-Typed>             ":" Type  & <Optional+>(                     "?=" Expression<+Block><-Break><-Return>)
-	|                <Named+>(Word "=")                     DestructureVariables                 <+Typed>                       & <Optional+>(                     "?=" Expression<+Block><-Break><-Return>)
+	|                <Named+>(Word "=")                     DestructureVariables<-Break><-Return><-Typed>             ":" Type  & <Optional+>(                     "?=" Expression<+Block><-Break><-Return>)
+	|                <Named+>(Word "=")                     DestructureVariables<-Break><-Return><+Typed>                       & <Optional+>(                     "?=" Expression<+Block><-Break><-Return>)
;
```
