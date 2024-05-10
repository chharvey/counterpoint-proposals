## Problem Statement
Given a variable of union type `[value: int] | [message: str]`, we might want a way to access the `.value` property of the left side, if it exists, else the `.message` property of the right side, else `null`.
```cp
claim result: [value: int] | [message: str];
if result.value then { %> TypeError: Property `value` does not exist on type `[value: int] | [message: str]`
	print.(result.value); %> TypeError
} else {
	throw result.message; %> TypeError
};
```
Because `.value` is not a property on *every* component of the union `[value: int] | [message: str]`, the compiler throws a TypeError for attempting to access `result.value`. The same is true for `result.message`. This is by design — we want the compiler to warn the developer for attempting to access a property that might not exist.

But we also get the same TypeError when attempting to use optional access.
```cp
claim result: [value: int] | [message: str];
if result?.value then { %> TypeError: Property `value` does not exist on type `[value: int] | [message: str]`
	print.(result?.value); %> TypeError
} else {
	throw result?.message; %> TypeError
};
```

## Possible Workarounds
No workarounds. There is currently no way to check if a property exists on an object before accessing it.

## Proposed Solution
The restriction that a property exist on *every* component of a union is relaxed for optional access: if the property exists on *some* union component, then its type is returned, but unioned with `null`.
```cp
claim result: [value: int] | [message: str];
result.value;  % regular access: still a TypeError
result?.value; % optional access: previously a TypeError, now type `int?`
```
Thus, the name of the “optional access” operator `?.` should be changed to **“potential access”** — seeing as it may be used to access non-optional properties that potentially exist.

With the restriction lifted, the produced *value* of the operator is the value on the object if it exists, else `null`.
```cp
let result1: [value: int] | [message: str] = [value= 42];
let i: int? = result1?.value;   %== 42
let s: str? = result1?.message; %== null

let result2: [value: int] | [message: str] = [message= "error!"];
let i: int? = result2?.value;   %== null
let s: str? = result2?.message; %== "error!"
```

Of course, if *none* of the union components have the potentially-accessed property, a TypeError is still thrown.
```cp
claim result: [value: int] | [message: str];
result.val;  % regular access: still a TypeError
result?.val; % potential access: still a TypeError
```

If *multiple* components of the union have the same property name, their types are unioned.
```cp
claim result: [isInt: true, value: int] | [isFloat: true, value: float] | [message: str];
result.value;                         %> TypeError
let v: int? | float? = result?.value; % produces the value at `result.value` if it exists, else `null`
```

This proposed change is compatible with nullable object access (an object whose type is a union with `null`). Instead of throwing a TypeError, the property’s type is itself unioned with `null`.
```cp
claim result: [value: int]?;
result.value;  %> TypeError
result?.value; %: int?
result.val;    %> TypeError
result?.val;   %> TypeError
```

One special note: If the binding object is `null` at runtime and access is attempted by expression (with brackets), the expression is *not evaluated*. This is known as ”short-circuiting”.
```cp
function return_0(): int {
	print.("I am returning 0.");
	return 0;
};
let result: [int]? = null;
result?.[return_0.()]; % does not evaluate `return_0.()` (and does not execute the print statement)
```
This is akin to short-circuiting in logical operators: in `false && return_0.()`, the function `return_0.()` is not called.

### Claim Access
The **claim access** operator may now be used for potential properties. As before, it subtracts `null` from the asserted type of the property.
```cp
claim result: [value: int] | [message: str];

result.value;  % regular access: still a TypeError
result?.value; % potential access: now type `int?`
result!.value; % claim access: previously a TypeError, now type `int` (subtracts `null` from potential access)

result.val;  % regular access: still a TypeError
result?.val; % potential access: still a TypeError
result!.val; % claim access: still a TypeError

let result1: [value: int] | [message: str] = [value= 42];
let i: int = result1!.value;   %== 42
let s: str = result1!.message; % type `str`, but `null` at runtime

let result2: [value: int] | [message: str] = [message= "error!"];
let i: int = result2!.value;   % type `int`, but `null` at runtime
let s: str = result2!.message; %== "error!"
```

Claim access only makes an assertion to the type-checker; it does not affect the compmiled output. In other words, `a!.b` compiles to the same output as `a.b`. In the example below, the last line evaluates `return_0.()` (and executes the print statement), but ultimately results in a runtime error, since `result` is `null`.
```cp
function return_0(): int {
	print.("I am returning 0.");
	return 0;
};
let result: [int] | void = null;
result!.[return_0.()]; % NullError at runtime
```

### Compatibility
Is this feature a “breaking” change? In other words, if this feature is introduced, will users’ existing code break and need to be changed/updated as a result? (Select only one.)
- ( ) yes, this feature is “breaking”
- (x) no, this feature is “non-breaking”
- ( ) not sure

### Benefits
Lifting the TypeError allows more expressive and concise code, and the ability to test whether potential properties exist in an object.

### Drawbacks
Possibility of not having a TypeError where needed.

### Details
The algorithm for type-checking is roughly as follows.
1. If `A` is not a union type, then the type of `a?.b` is just the type of `a.b`, that is, (assuming `a` is of type `A`) type `A.b` if it exists, otherwise throw a TypeError. If `a` is just type `null`, then `a?.b` throws a TypeError.
1. If `A` is a union type `A1 | A2 | A3`, then to get the type of `a?.b`: If at least one component of `A` has a `.b` property, map each component `A‹n›` of `A` to type `A‹n›.b` if it exists, else `null`, and then union those all. (This applies if any component is `null` as well.) Otherwise, throw a TypeError.

The algorithm for evaluation is basically the same as before.
1. If `a.b` exists, then the value of `a?.b` at runtime is `a.b` (even if `a.b` is `null`);
1. If `a.b` does not exist, then `a?.b` produces `null`.

The algorithm for expression bracket access adds the following short-circuit:
1. If `a` is `null`, then `a?.[‹expr›]` *does not evaluate `‹expr›`*, and then produces `null`.

#### Potential Function Calls
This section will not be developed; it is only a design discussion.

The **potential function call** `fn?.(‹args›)` can be thought of as the potential access of a property of `fn` — think of `fn?.(‹args›)` as something like `fn?.call`.
1. If `fn` is callable and *only* callable, then the type of `fn?.(‹args›)` is the type of `fn.(‹args›)` (assuming the arguments are type-valid).
1. If `fn` is not at all callable, including the case that `fn` is equal to `null`, then a TypeError is thrown.
1. If `fn` is the union of a callable type and a non-callable type (including `null`), then the type of `fn?.(‹args›)` is the type of `fn_c.(‹args›)` (again, assuming valid arguments, and `fn_c` being the “callable part” of `fn`) unioned with `null`.

Runtime evaluation:
1. If `fn` is callable, then `fn?.(‹args›)` produces the result of calling `fn` with the evaluated `‹args›`.
1. If `fn` is not callable, then `fn?.(‹args›)` *does not evaluate `‹args›`* (short-circuiting), and then produces `null`.

## Alternatives
One alternative solution would be to have a new operator that differs from optional access just for the union case, but we already have three types of access operators. Another solution could be to add an operator that tests keys and returns boolean, a la `if .value in result then … else …`, but use cases would be limited; the potential access operator is more versatile, e.g., `print.(result?.value || result?.message)`.
