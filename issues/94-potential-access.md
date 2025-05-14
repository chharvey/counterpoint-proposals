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
This is because a TypeErrorNoEntry is thrown even before checking the kind of access (`.` or `?.` or `!.`) that was used.

## Possible Workarounds
No workarounds. There is currently no way to check if a property exists on an object before accessing it.

## Proposed Solution
The restriction that a property exist on *every* component of a union is relaxed for optional access: if the property exists on *some* union component, then its type is returned, but unioned with `null`.
```cp
claim result: [value: int] | [message: str];
result.value;  % regular access: still a TypeError
result?.value; % optional access: previously a TypeError, now type `int?`
```
Thus, the name of the “optional access” operator `?.` should be changed to **“maybe access”** — seeing as it may be used to access non-optional properties that potentially exist. In Version 0.5 it will be overloaded to work with the Maybe type.

With the restriction lifted, the produced *value* of the operator is the value on the object if it exists, else `null`.
```cp
let result1: [value: int] | [message: str] = [value= 42];
let i: int? = result1?.value;   %== 42
let s: str? = result1?.message; %== null

let result2: [value: int] | [message: str] = [message= "error!"];
let i: int? = result2?.value;   %== null
let s: str? = result2?.message; %== "error!"
```

Of course, if *none* of the union components have the maybe-accessed property, a TypeError is still thrown.
```cp
claim result: [value: int] | [message: str];
result.val;  % regular access: still a TypeError
result?.val; % maybe access: still a TypeError
```
A special case of this is if the binding object’s type is `null` or narrower.
```cp
claim result: null;
result.value;  %> TypeError
result?.value; %> TypeError
result.val;    %> TypeError
result?.val;   %> TypeError
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

### Short-Circuiting
This section propose a new feature. If the binding object is `null` at runtime, and maybe access is used with bracketed expression syntax, then the bracketed expression is *not evaluated*. This is known as **short-circuiting**.
```cp
function return_0(): int {
	print.("I am returning 0.");
	return 0;
};
let result: int[]? = null;
let access_regular: int  = result.[return_0.()];  %> TypeError
let access_maybe:   int? = result?.[return_0.()]; % safe, and does not evaluate `return_0.()` (and does not execute the print statement)
```
Regular access throws an error, but for maybe access, the bracketed expression `return_0.()` is not executed. The resulting value of the whole expression is `null`. This is akin to short-circuiting in logical operators: in `anything && return_0.()`, the call `return_0.()` might not executed based on the runtime value of `anything`.

### Optional Entries
The behavior of the newly-named maybe access operator hasn’t changed when accessing an optional entry of a static type (tuple/record): the type of the entry will still be unioned with `null`.

This section proposes two new features for static types.

First, when accessing an optional entry on a static type, the maybe access operator is mandatory. Using regular access will result in a compile-time TypeError. (Previously, the access was allowed and the expression’s type was unioned with `void`.)
```cp
let var person: [name: str, nickname?: str] = [name= "Thomas Jefferson", nickname= "Tom"];
person.nickname;  %> TypeError % an optional property requires the maybe access operator
person?.nickname; % type `str?`, equals `"Tom"`

set person = [name= "John Adams"];
person?.nickname; % type `str?`, equals `null`
```

Second, the inverse is true. If using the *maybe access* operator on a *required entry*, a TypeError will be thrown. (Previously, the potental access was allowed but it did not change the expression’s type.) Now the use of the regular access operator is strictly enforced.
```cp
let person: [name: str, nickname?: str] = [name= "Thomas Jefferson", nickname= "Tom"];
person?.name; %> TypeError % a required property requires the regular access operator
person.name;  % ok, type `str`
```

For unions of static types, if a property is missing or optional on any component, we consider it to be optional and the maybe access operator is needed. Only when every component requires the property may the regular access operator be used.
```cp
let result: [value: int, isError: bool, since?: float] | [message: str, isError: bool, since: float] = [value= 42, isError= false];
% notice `since` is optional in the first comonent but required in the second --- this makes it optional overall
result?.isError; %> TypeError % a required property requires the regular access operator
result.isError;  % ok
result?.since;   % ok
result.since;    %> TypeError % an optional property requires the maybe access operator
```

For dynamic types (e.g. lists/dicts), both the regular and maybe access operators are allowed. Regular access returns the invariant type of the data structure, whereas optional access unions it with `null`.

### Claim Access
The **claim access** operator `!.` will be dropped and unsupported until Version 0.5. It will be used for Result types and Exceptions. Though still allowed by syntax, it is now temporarily a semantic error to use them.
```cp
result!.value; %> SemanticError
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
1. If `A` is *not* a union type, assuming `a` is of type `A`,
	1. If `A` is a static type (tuple/record/primitive type):
		1. If type `A.b` exists and is not optional: the type of `a.b` is `A.b`. The expression `a?.b` throws a TypeError.
		1. If type `A.b` exists and is     optional: the type of `a?.b` is `A.b | null`. The expression `a.b` throws a TypeError.
		1. If type `A.b` does not exist (including the case that `A` is a subtype of `null`), then both `a.b` and `a?.b` throw TypeErrors.
	1. If `A` is a dynamic type (list/dict/set/map/other object):
		1. The type of `a.b` is `A.b`.
		1. The type of `a?.b` is `A.b | null`.
1. If `A` is a union type `A1 | A2 | A3` (consider type `unknown` to be an infinite union), assuming `a` is of type `A`,
	1. To get the type of `a.b`
		1. If `A` is `unknown`, throw a TypeError.
		1. If *every* component `A‹n›` of `A` has a `.b` property and it is required, then return the union of all the types of `A‹n›.b` using the rules above.
		1. Otherwise, throw a TypeError. (This also applies if any component is `null`, or if any `A‹n›.b` exists but is optional.)
	1. To get the type of `a?.b`:
		1. If `A` is `unknown`, return type `unknown`.
		1. If *some* component of `A` has a `.b` property, required or optional, map each component `A‹n›` of `A` to type `A‹n›.b` if it exists, else `null`, and then return the union of those all. (This applies if any component is `null` as well.)
		1. Otherwise, throw a TypeError.

The algorithm for evaluation is basically the same as before, with the additional short-circuit for null-checking in `?.`.
1. Evaluating `a.b`:
	1. Evaluate `a`.
	1. If `a` has a property `b`, return the value of `a.b` (even if it is `null`).
	1. If `a` does not have a property `b`, throw a runtime error.
1. Evaluating `a?.b`:
	1. Evaluate `a`.
	1. If `a` is `null`, return `null`.
	1. Try to evaluate `a.b` and return it.
	1. If an error was caught, return `null`.
1. Evaluating `a.[b]`:
	1. Evaluate `a`.
	1. Evaluate `b`.
	1. If `b` is within the bounds/range of `a`, return the value of `a.[b]` (even if it is `null`).
	1. If `b` is out of ounds/range of `a`, throw a runtime error.
1. Evaluating `a?.[b]`:
	1. Evaluate `a`.
	1. If `a` is `null`, return `null`.
	1. Evaluate `b`.
	1. Try to evaluate `a.[b]` and return it.
	1. If an error was caught, return `null`.

#### Potential Function Calls
This section will not be developed; it is only a design discussion.

The **potential function call** `fn?.(‹args›)` can be thought of as the maybe access of a property of `fn` — think of `fn?.(‹args›)` as something like `fn?.call`.
1. If `fn` is callable and *only* callable, then the type of `fn?.(‹args›)` is the type of `fn.(‹args›)` (assuming the arguments are type-valid).
1. If `fn` is *not at all* callable, including the case that `fn` is equal to `null`, then a TypeError is thrown.
1. If `fn` is the union of a callable type and a non-callable type (including `null`), then the type of `fn?.(‹args›)` is the type of `fn_c.(‹args›)` unioned with `null` (assuming valid arguments, and `fn_c` being the “callable part” of `fn`).
1. Type `unknown` can be thought of as an infinite union, so if `fn` is type `unknown`, then `fn?.(‹args›)` is of type `unknown` (while `fn.(‹args›)` would be a TypeError).

Runtime evaluation:
1. Evaluating `fn.(‹args›)`:
	1. Evaluate `fn`.
	1. Evaluate `‹args›`.
	1. Call `fn` with `‹args›` and return the result. (If the call throws, bubble up the error.)
1. Evaluating `fn?.(‹args›)`:
	1. Evaluate `fn`.
	1. If `fn` is `null`, return `null`.
	1. If `fn` is otherwise not callable, return `null`.
	1. Evaluate `‹args›`.
	1. Return the evaluation of `fn.(‹args›)`.

## Alternatives
One alternative solution would be to have a new operator that differs from optional access just for the union case, but we already have three types of access operators. Another solution could be to add an operator that tests keys and returns boolean, a la `if @value in result then … else …`, but use cases would be limited; the maybe access operator is more versatile, e.g., `print.(result?.value || result?.message)`.
