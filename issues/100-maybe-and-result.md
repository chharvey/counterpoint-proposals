The built-in types `Maybe<T>` and `Result<T, E>` declare values that may be one type or another. They correspond to `T | null` and `T | Exception` respectively, but with major differences.

## Maybe
The built-in type `Maybe<T>` declares a value that may or may not exist. It is similar to `T | null` except that `Maybe` is a **discriminated** (“tagged”) union, or “algebraic sum” type. Unlike a (“untagged”) type union, it has structure. The `Maybe` type is split into two subtypes, `Some` and `None`. The `Some` type has an internal value whereas `None` does not.
```cp
interface Maybe<out T> {
	then<U ?= T>(callback: (T) => U): Maybe.<U>;
}
interface Some<out T> inherits Maybe.<T> {
	new (value: T);
	% private readonly value: T;
}
interface None<out T> inherits Maybe.<T> {
	new ();
}
```

`Maybe` is stricter than a type union because it lets programmers reason about objects based on their construction, rather than their properties.
```cp
let var x: Maybe.<int> = Some.<int>(42);
let y: Maybe.<int> = x.then.((value) => value + 1);
assert y == Some.<int>(43);

set x = None.<int>();
let z: Maybe.<int> = x.then.((value) => value + 1);
assert z == None.<int>();
assert z !== x; % `.then.()` always returns a new Maybe reference type object
```

The `Maybe` type is *not* the same as an untagged union `T | null`. Neither `T` nor `null` are directly assignable to it. Instead, we construct a `Some` or `None`.
```cp
let x: Maybe.<int> = 42;   %> TypeErrorNotAssignable
let y: Maybe.<int> = null; %> TypeErrorNotAssignable

let x: Maybe.<int> = Some.<int>(42); % ok
let y: Maybe.<int> = None.<int>();   % ok

if x is Integer then {}; % will always be false
if y == null then {};    % will always be false
```

What are the benefits of `Maybe` over a union?
```cp
let var union: int | null = 42;
set union += 1; %> TypeErrorInvalidOperation
if union != null then {
	set union += 1; % ok
};
match union {
	Null    -> print.("none"),
	Integer -> print.(union),
};

let var maybe: Maybe.<int> = Some.<int>(42);
set maybe += 1; %> TypeErrorInvalidOperation
if maybe is Some then {
	set maybe += 1;                                %> TypeErrorInvalidOperation
	set maybe  = maybe.map.((value) => value + 1); % ok
	set maybe  = Some.<int>(maybe~? + 1);          % better
};
% pattern-matching/unwrapping
if maybe is Some.<int>(let x) then {
	set maybe += 1;                 %> TypeErrorInvalidOperation
	set maybe  = Some.<int>(x + 1); % best
};
match maybe {
	None.<int>()      -> print.("none"),
	Some.<int>(let x) -> print.(x),
};
```

### Type Shorthand and The Maybe Access Operator
Type `T?` is shorthand for `Maybe.<T>`. It creates a discriminated union of its type parameter and nothing. It is *not* shorthand for the union `T | null`.
```cp
let rec_maybe: [item: int]? = Some.<[item: int]>([item= 42]);
%              ^ shorthand for `Maybe.<[value: int]>`
```

The **maybe access operator** `?.` is overloaded to work with the `Maybe` type. It returns a new `Maybe` that wraps the value if it is not a `None`, else it returns the same `None`.
```cp
rec_maybe.item;  %> TypeErrorNoEntry
rec_maybe?.item; % equivalent to `rec_maybe.then.((val) => val.item)`
assert rec_maybe?.item == Some.<int>(42);

let tup_maybe: [int]? = None.<[int]>();
assert tup_maybe?.0 == None.<int>();
assert tup_maybe?.0 === tup_maybe; % returns the same Maybe reference type object
```

Short-circuiting: When the operand of `?.` is `null` or a `None`, and the accessor is an expression via brackets, the accessor is *not evaluated*. In the example below, `return_0.()` is never evaluated, and nothing is printed.
```cp
function return_0(): int {
	print.("I am returning 0.");
	return 0;
};
let value_u: [int] | null = null;
value_u?.[return_0.()]; % returns `value_u`, does not execute function

let value_m: [int]? = None.<[int]>();
value_m?.[return_0.()]; % returns `value_m`, does not execute function
```

### Non-Null Assertion
The **non-null assertion** operator `~?` has two functions. For most values, it subtracts `null` from the type of its operand: It’s equivalent to `expr as <T>`, where `expr` is of type `T | null`. However, if the operand is a `Maybe.<T>` type, then it performs an additional function at runtime: it ‘unwraps’ the value of the Maybe, returning type `T`. Additionally, if the value is actually `null` or a `None` at runtime, the `~?` operator throws an error.
```cp
claim rec_union: [item: int] | null;

let rec: [item: int] = rec_union;   %> TypeError
let rec: [item: int] = rec_union~?; % no TypeError, but unsafe!
if !!rec_union then {
	let rec: [item: int] = rec_union~?; % no TypeError, and safe
};

rec_union.item;                    %> TypeErrorNoEntry
rec_union?.item;                   % maybe access: returns type `int | null`
(rec_union as <[item: int]>).item; % claims `rec_union` is type `[item: int]`; returns type `int`
rec_union~?.item;                  % shorthand for the expression claim above, but throws at runtime if `rec_union` is `null`

claim rec_maybe: Maybe.<[item: int]>;

let rec: [item: int] = rec_maybe;   %> TypeError
let rec: [item: int] = rec_maybe~?; % no TypeError, but unsafe!
if rec_maybe is Some then {
	let rec: [item: int] = rec_maybe~?; % no TypeError, and safe
};

rec_maybe.item;                    %> TypeErrorNoEntry
rec_maybe?.item;                   % maybe access: returns type `Maybe.<int>`
(rec_maybe as <[item: int]>).item; %> TypeError: `Maybe.<[item: int]>` cannot be narrowed to `[item: int]` since they have no overlap
rec_maybe~?.item;                  % asserts `rec_maybe` is `[item: int]`; returns type `int`; but throws at runtime if `rec_maybe` is a `None`
```

Nested `Maybe`s *do not* automatically flatten. That is, a `Maybe.<Maybe.<T>>` is not the same as a `Maybe.<T>`. Accordingly, the non-null assertion operator `~?` only unwraps a singular `Maybe`. (Keep in mind, these are rare cases; `Maybe`s should not typically be nested.)
```cp
let var nested: int?? = None.<Maybe.<int>>();
%               ^ shorthand for `Maybe.<Maybe.<int>>`
let var value: unknown = 0;
if nested is Some then {
	set value = nested~?;
};
assert value == 0; % `nested` is not a `Some`, so nothing changes here

set nested = Some.<Maybe.<int>>(None.<int>());
if nested is Some then {
	set value = nested~?;
};
assert value == None.<int>(); % `nested` is a `Some` whose wrapped value is a `None`

set nested = Some.<Maybe.<int>>(Some.<int>(1));
if nested is Some then {
	set value = nested~?;
};
assert value == Some.<int>(1); % `nested` is a `Some` whose wrapped value is also a `Some`

set value = value~?; % unsafe, but we are sure `value` is not a `None`
assert value == 1;
```

### Guidance
For accessors, `maybe?.accessor` and `maybe~?.accessor` have different use cases. Use `maybe?.accessor` when `maybe` could be a value or `null`, or it’s a `Maybe` object and you don’t know what the branch will be at runtime. Use `maybe~?.accessor` when you know for sure that `maybe` is not `null` and it is not a `None` branch at runtime, and it definitely has an `.accessor` property, based on external factors or business logic.

For operating on `Maybe` objects in general, using `?.` is not possible, so use conditionals in conjunction with `~?`.
```cp
return maybe == "foo"; % probably a mistake!

if maybe is Some then {
	return maybe~? == "foo"; % fixed
};
```

## Result
The built-in type `Result<T, E>` declares a value that may be any value or may be an Exception. It is similar to `T | Exception` except that `Result` is a **discriminated** (“tagged”) union, or “algebraic sum” type. Unlike a (“untagged”) type union, it has structure. The `Result` type is split into two subtypes, `Ok` and `Fail`. Both subtypes have internal values, with the value of `Fail` being an instance of the `Exception` class.
```cp
interface Result<out T, out E narrows Exception ?= Exception> {
	then<U ?= T, V ?= E>(on_ok: (T) => U, on_ex: (E) => V): Result.<U, V>;
}
interface Ok<out T, out E narrows Exception ?= Exception> inherits Result.<T, E> {
	new (value: T);
	% private readonly value: T;
}
interface Fail<out T, out E narrows Exception ?= Exception> inherits Result.<T, E> {
	new (reason?: E | str);
	% private readonly reason: E;
}
```

`Result` is stricter than a type union because it lets programmers reason about objects based on their construction, rather than their properties.
```cp
let var x: Result.<int> = Ok.<int>(42);
let y: Result.<int> = x.then.((value) => value + 1);
assert y == Ok.<int>(43);

set x = Fail.<int>();
let z: Result.<int> = x.then.((value) => value + 1);
assert z == Fail.<int>();
assert z !== x; % `.then.()` always returns a new Result reference type object
```

The `Result` type is *not* the same as an untagged union `T | Excetion`. Neither `T` nor `Exception` are directly assignable to it. Instead, we construct an `Ok` or `Fail`.
```cp
let x: Result.<int> = 42;                  %> TypeErrorNotAssignable
let y: Result.<int> = Exception.("oops!"); %> TypeErrorNotAssignable

let x: Result.<int> = Ok.<int>(42);        % ok
let y: Result.<int> = Fail.<int>("oops!"); % ok

if x is Integer then {};   % will always be false
if y is Exception then {}; % will always be false
```

What are the benefits of `Result` over a union?
```cp
let var union: int | Exception = 42;
set union += 1; %> TypeErrorInvalidOperation
if union isnt Exception then {
	set union += 1; % ok
};
match union {
	Exception -> print.("err"),
	Integer   -> print.(union),
};

let var result: Result.<int> = Ok.<int>(42);
set result += 1; %> TypeErrorInvalidOperation
if result is Ok then {
	set result += 1;                                 %> TypeErrorInvalidOperation
	set result  = result.map.((value) => value + 1); % ok
	set result  = Ok.<int>(result~! + 1);            % better
};
% pattern-matching/unwrapping
if result is Ok.<int>(let x) then {
	set result += 1;               %> TypeErrorInvalidOperation
	set result  = Ok.<int>(x + 1); % best
};
match result {
	Fail.<int>()    -> print.("err"),
	Ok.<int>(let x) -> print.(x),
};
```

### Type Shorthand and The Result Access Operator
Type `T!` is shorthand for `Result.<T>`. It creates a discriminated union of its type parameter and an Exception type. It is *not* shorthand for the union `T | Exception`.
```cp
let rec_result: [item: int]! = Ok.<[item: int]>([item= 42]);
%               ^ shorthand for `Result.<[value: int]>`
```

The **result access operator** `!.` is overloaded to work with the `Result` type. It returns a new `Result` that wraps the value if it is not a `Fail`, else it returns the same `Fail`.
```cp
rec_result.item;  %> TypeErrorNoEntry
rec_result!.item; % equivalent to `rec_result.then.((val) => val.item)`
assert rec_result!.item == Ok.<int>(42);

let tup_result: [int]! = Fail.<[int]>();
assert tup_result!.0 == Fail.<int>();
assert tup_result!.0 === tup_result; % returns the same Result reference type object
```

Short-circuiting: When the operand of `!.` is an `Exception` or a `Fail`, and the accessor is an expression via brackets, the accessor is *not evaluated*. In the example below, `return_0.()` is never evaluated, and nothing is printed.
```cp
function return_0(): int {
	print.("I am returning 0.");
	return 0;
};
let value_u: [int] | Exception = Exception.("oops!");
value_u!.[return_0.()]; % returns `value_u`, does not execute function

let value_r: [int]! = Fail.<[int]>("oops!");
value_r!.[return_0.()]; % returns `value_r`, does not execute function
```

### Non-Exception Assertion
The **non-exception assertion** operator `~!` has two functions. For most values, it subtracts `Exception` from the type of its operand: It’s equivalent to `expr as <T>`, where `expr` is of type `T | Exception`. However, if the operand is a `Result.<T, E>` type, then it performs an additional function at runtime: it ‘unwraps’ the value of the Result, returning type `T`. Additionally, if the value is actually an `Exception` or a `Fail` at runtime, the `~!` operator throws the exception.
```cp
claim rec_union: [item: int] | Exception;

let rec: [item: int] = rec_union;   %> TypeError
let rec: [item: int] = rec_union~!; % no TypeError, but unsafe!
if !!rec_union then { % (remember, Exceptions are falsy)
	let rec: [item: int] = rec_union~!; % no TypeError, and safe
};

rec_union.item;                    %> TypeErrorNoEntry
rec_union!.item;                   % result access: returns type `int | Exception`
(rec_union as <[item: int]>).item; % claims `rec_union` is type `[item: int]`; returns type `int`
rec_union~!.item;                  % shorthand for the expression claim above, but throws at runtime if `rec_union` is an `Exception`

claim rec_result: Result.<[item: int]>;

let rec: [item: int] = rec_result;   %> TypeError
let rec: [item: int] = rec_result~!; % no TypeError, but unsafe!
if rec_result is Ok then {
	let rec: [item: int] = rec_result~!; % no TypeError, and safe
};

rec_result.item;                    %> TypeErrorNoEntry
rec_result!.item;                   % result access: returns type `Result.<int>`
(rec_result as <[item: int]>).item; %> TypeError: `Result.<[item: int]>` cannot be narrowed to `[item: int]` since they have no overlap
rec_result~!.item;                  % asserts `rec_result` is `[item: int]`; returns type `int`; but throws at runtime if `rec_result` is a `Fail`
```

Nested `Result`s *do not* automatically flatten. That is, a `Result.<Result.<T>>` is not the same as a `Result.<T>`. Accordingly, the non-exception assertion operator `~!` only unwraps a singular `Result`. (Keep in mind, these are rare cases; `Result`s should not typically be nested.)
```cp
let var nested: int!! = Fail.<Result.<int>>("reason 0");
%               ^ shorthand for `Result.<Result.<int>>`
let var value: unknown = 0;
if nested is Ok then {
	set value = nested~!;
};
assert value == 0; % `nested` is not an `Ok`, so nothing changes here

set nested = Ok.<Result.<int>>(Fail.<int>("reason 1"));
if nested is Ok then {
	set value = nested~!;
};
assert value == Fail.<int>("reason 1"); % `nested` is an `Ok` whose wrapped value is a `Fail`

set nested = Ok.<Result.<int>>(Ok.<int>(1));
if nested is Ok then {
	set value = nested~!;
};
assert value == Ok.<int>(1); % `nested` is an `Ok` whose wrapped value is also an `Ok`

set value = value~!; % unsafe, but we are sure `value` is not a `Fail`
assert value == 1;
```

### Guidance
For accessors, `result!.accessor` and `result~!.accessor` have different use cases. Use `result!.accessor` when `result` could be a value or an `Exception`, or it’s a `Result` object and you don’t know what the branch will be at runtime. Use `result~!.accessor` when you know for sure that `result` is not an `Exception` and it is not a `Fail` branch at runtime, and it definitely has an `.accessor` property, based on external factors or business logic.

For operating on `Result` objects in general, using `!.` is not possible, so use conditionals in conjunction with `~!`.
```cp
return result == "foo"; % probably a mistake!

if result is Ok then {
	return result~! == "foo"; % fixed
};
```

## Logical Operator Overloads
### Logical Negation and Is-Empty
The logical negation `!` and “is-empty” `?` unary operators are overloaded to work well with the Maybe and Result types. If `x` is a None variant of Maybe, then it is falsy (and thus “empty”), thus `!x` and `?x` are both `true`. If `y` is a Fail variant of Result, then it is falsy (and thus “empty”), thus `!y` and `?y` are both `true`. This aligns with the current behavior of Exception objects.
```cp
let ex: Exception = Exception.("message");
!ex; %== true
?ex; %== true

let var x: str? = Some.<str>("value");
!x; %== false
?x; %== false

set x = None.<str>();
!x; %== true
?x; %== true

let var y: str! = Ok.<str>("value");
!y; %== false
?y; %== false

set y = Fail.<str>("reason");
!y; %== true
?y; %== true
```

### Logical Conjunction and Disjunction
The logical conjunction `&&` and logical disjunction `||` binary operators are overloaded to work well with the Maybe and Result types.

As a review: logical conjunction (`&&`) returns the operand on the left-hand side (LHS) if it is falsy and skips evaluation of the operand on the right-hand side (RHS); otherwise, it evaluates the RHS and returns it. Logical disjunction (`||`) returns the LHS if it is truthy and skips evaluation of the RHS; otherwise it evaluates and returns the RHS.

Besides Exception objects, None objects, and Fail objects, all other objects are always truthy. Exception, None, and Fail objects are treated as falsy by design (even though they are objects).
```cp
claim rhs: unknown;

% the null value
null && rhs; %== null
null || rhs; %== rhs

% falsy values
false && rhs; %== false
false || rhs; %== rhs
% Exceptions
Exception.("demo") && rhs; %== Exception.("demo")
Exception.("demo") || rhs; %== rhs
% None objects
None.<unknown>() && rhs; %== None.()
None.<unknown>() || rhs; %== rhs
% Fail objects
Fail.<unknown>("demo") && rhs; %== Fail.("demo")
Fail.<unknown>("demo") || rhs; %== rhs

% empty values (if not falsy) are truthy
"" && rhs; %== rhs
"" || rhs; %== ""
[] && rhs; %== rhs
[] || rhs; %== []

% non-empty values are always truthy
"hello" && rhs; %== rhs
"hello" || rhs; %== "hello"


% Maybe objects
let var x: str? = Some.<str>("value");
x && rhs; %== rhs
x || rhs; %== x

set x = None.<str>();
x && rhs; %== x
x || rhs; %== rhs

% Result objects
let var y: str! = Ok.<str>("value");
y && rhs; %== rhs
y || rhs; %== y

set y = Fail.<str>("reason");
y && rhs; %== y
y || rhs; %== rhs
```

## Maybe and Result are Abstractions
The `Maybe.<T>` and `Result.<T, Exception>` types are *abstractions* of the type unions `T | null` and `T | Exception` respectively. It is still possible to have a `Some.<null>` and an `Ok.<Exception>`, and they are treated as truthy just like all other `Some`s and `Ok`s. What matters isn’t the internal value, but the variant of the class.
```cp
!Some.<null>(null);                     %== false
!Ok.<Exception>(Exception.("message")); %== false
```
