Remove the Void type.

# Introduction
Previously, `void` was a type, but no value for it existed. That was a weird thing to do, since types are supposed to represent sets of values. `void` was a type with no value, but it was distinct from the `never` type. While the latter is truly the “empty” or “bottom” type — it contains no values, is accepted where every other type is accepted, and anything could be assumed about it — the same can not be said about the former. Though `void` contained no values, it was not accepted where other types were accepted (unless those types were unioned with `void`), and we could not assume anything about expressions of type `void`. Thus, the Counterpoint language had treated `void` as a “pseudo-type”, by making special exceptions for it.

This issue solves those discrepancies by simply removing the Void type and replacing it with the Null type in most situations. The keyword `void` will still be allowed in function signatures, but it does not represent an actual Type, which abides by all the rules and operations of type theory.

The default value of uninitialized variables/parameters and unassigned optional entries of tuples/records is now the value `null`.

# Changes

## BREAKING: Keyword `void` Is Not A Type
Void is no longer a type, and cannot be treated as types in the type system. The keyword `void` is still reserved, but may not be used.
```cp
type T = void;             %> SyntaxError % breaking change!
let a: float | void = 4.2; %> SyntaxError % breaking change!

type void = float | int; %> SyntaxError % still an error
let void: float = 4.2;   %> SyntaxError % still an error
```

## Variable Declaration
Variables may now be declared without an excplicit initial value. Such variables are initialized to `null` (and thus are unioned with `null`).
```cp
let var greeting?: str;
```
This new syntax declares `greeting` as an unitialized variable, with an implicit initial value of `null`. When accessed, its type is `str?`, but it may only be assigned `str` values. (As a reminder, the type operation `str?` represents the union `str | null`.)
```cp
let var greeting?: str;
let greet_1: str  = greeting; %> TypeError: Expression of type `str | null` is not assignable to type `str`.
let greet_2: str? = greeting; % ok

set greeting = null;    %> TypeError: Expression of type `null` is not assignable to type `str`.
set greeting = "hello"; % ok
```
The read type of `greeting` is `str?`, but its write type is just `str`.

It’s a syntax error for an unassigned variable to be declared fixed — it requires the `var` modifier.
```cp
let greeting?: str; %> ParseError
```

There’s a subtle difference between the statements `let var greeting: str? = "hello";` and `let var greeting?: str;`, besides the former being initialized. The former says that a value for `greeting` always exists, and whenever accessed or assigned, that value’s type is the union `str | null`. The latter says that a value for `greeting` might or might not exist, but if it does exist, its type is definitely `str` and not `null`. When assigned, it only accepts values of type `str`.

This patten is similar to optional tuple/record entries. The entry might not exist, so accessing it unions the type with `null`, but when assigned, only values of the declared type are assignable.
```cp
let greetings: [?: str] = [];
greetings.0;                  %: str?
set greetings.0 = "a string"; %: str
```

Issue #55 is updated to include uninitialized optional parameters.
```cp
function moveForward(var steps?: int): void {
	steps;            %: int?
	set steps = 3;    %: int
}
```

## Out-Of-Bounds and Out-Of-Range Access
As before, when using regular access on static types like tuples/records, the typer will report a TypeError if the accessor does not exist on the binding object. If that type-checking is bypassed (e.g. via a type claim), then some other sort of error may result.

If both the binding object and accessor are foldable, then the compiler wil report an TypeErrorIndexOutOfBounds or TypeErrorKeyOutOfRange (for tuples or records, respectively) for regular access. This was previously reported as a VoidError.

If however the bining object or accessor are *not* foldable, the compiler won’t report any errors; instead, the runtime will throw an IndexOutOfBoundsException or KeyOutOfRangeException respectively. This was previosly not implmented.

```cp
let list: int[]   = List.<int>([2, 3]);
let dict: [: int] = Dict.<int>([a= 2, b= 3]);

let x: int = list.[2];  %> TypeErrorIndexOutOfBounds % since both accessee and accessor are foldable
let y: int = dict.[@c]; %> TypeErrorKeyOutOfRange    % since both accessee and accessor are foldable

let var i = 2;
let var k = @c;
let xx: int = list.[i]; % no compiler error (index is not foldable), but unsafe: throws IndexOutOfBoundsException at runtime
let yy: int = dict.[c]; % no compiler error (key   is not foldable), but unsafe: throws KeyOutOfRangeException at runtime
```
(As an aside, it is not usually safe to access dynamic collections with static indices. The recommended technique is to loop over the collection dynamically.)

As decribed in #94, the potential access operator “catches” the runtime error and returns `null` instead.
```cp
let xx: int = list.[i]; % no compiler error (index is not foldable), but unsafe: throws IndexOutOfBoundsException at runtime
let yy: int = dict.[c]; % no compiler error (key   is not foldable), but unsafe: throws KeyOutOfRangeException at runtime
```

## Optional Entries
For static types (tuples, records), optional entries now have a default value of `null` when accessed.
```cp
let tup: [int, float, ?: str] = [0, 1.1]; % contains two values when constructed: `0` and `1.1`
let list: int[] = List.<int>([2, 3]);     % contains two values when constructed: `2` and `3`
```
If `a` is a static object with an optional property `b`, then regular access (`a.b`) is now typed as `B?` (previously it was `B | void`). Note that `B?` is also the resulting type via potential access (`a?.b`). At runtime this always produces the value of `a.b`, even if it’s `null`. (Previously accessing `a.b` would result in a VoidError.)
```cp
let s: str  = tup.2;   %> TypeError: `str | null` is not assignable to `str`
let s: str? = tup.2;   % ok, produces `null` at runtime
```

### Potential Access
Potential access (`a?.b`) remains basically the same. The value of `a?.b` at runtime is `a.b` if it exists, else `null`. For optional entries of static collections, the type of `a?.b` is still `B?`.
```cp
let tup: [int, float, ?: str] = [0, 1.1];
let list: int[] = List.<int>([2, 3]);

let tup1_r: float = tup.1; %== 1.1
let tup2_r: str?  = tup.2; % produces `null` at runtime

let tup1_o: float = tup?.1; % same as regular access
let tup2_o: str?  = tup?.2; % same as regular access

let list1_r: int = list.[1]; %== 3
let list2_r: int = list.[2]; % throws IndexOutOfBoundsException at runtime

let list1_o: int = list?.[1]; % same as regular access
let list2_o: int = list?.[2]; % same as regular access
```
As shown above, potential access is not very useful for objects of a single type, but we can still use it if the binding object could be `null` or, as introduced in #94, a union of collection types.
```cp
let voidable_rec?: [prop: str];
voidable_rec.prop;  %> TypeError: Property `prop` does not exist on type `[prop: str] | null`.
voidable_rec?.prop; %== `null`
```

### Claim Access
Claim access (`a!.b`) type remains as `B`, subtracting `null`. Useful for bypassing type errors.
```cp
let rec: [a: int, b?: str, c: float] = [a= 0, b= "hello", c= 1.1];
let y: str = rec.b;  %> TypeError: `str?` not assignable to `str`
let y: str = rec!.b; % ok
rec!.b;              %== "hello"
rec!.b.length;       %== 5

let tup: [int, float, ?: str] = [0, 1.1];
let y: str = tup.2;  %> TypeError: `str?` not assignable to `str`
let y: str = tup!.2; % ok, but unsafe
tup!.2;              %== `null`
tup!.2.length;       % error at runtime!
```

## NullError
`NullError` is a new class of runtime error that communicates an invalid access to a property on `null`.
```cp
let tup: [int, float, ?: str] = [0, 1.1];
tup!.2.length; % no error at compile time, but NullError at runtime
```
