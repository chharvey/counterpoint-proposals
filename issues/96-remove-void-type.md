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
set greetings.0 = null;       %> TypeError: Expression of type `null` is not assignable to type `str`.
set greetings.0 = "a string"; % ok
```

Issue #55 is updated to include uninitialized optional parameters.
```cp
function moveForward(var steps?: int): void {
	steps; %: int?
	set steps = null; %> TypeError
	set steps = 3;    % ok
}
```

## Optional Entries
For static types (tuples, records), optional entries now have a default value of `null`. The value exists in the object at the time of construction. For dynamic types (lists, dicts), if no value exists in the object at the access point, an IndexOutOfBoundsException will be thrown at runtime at the time of access.
```cp
let tup: [int, float, ?: str] = [0, 1.1]; % contains three values when constructed: `0`, `1.1`, and `null`
let list: int[] = List.<int>([2, 3]);     % contains only two values when constructed: `2` and `3`
```
If `a` is an object with an optional property `b`, then regular access (`a.b`) is still typed as `B?`. At runtime this always produces the value of `a.b`, even if it’s `null`.
```cp
let s: str  = tup.2;   %> TypeError: `str | null` is not assignable to `str`
let s: str? = tup.2;   % ok, produces `null` at runtime

let i: int = list.[2]; % ok, but throws IndexOutOfBoundsException at runtime
```

### Optional Access
Optional access (`a?.b`) remains basically the same. The value of `a?.b` at runtime is `a.b` if it exists, else `null`.
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
As shown above, optional access is not very useful for objects of a single type, but we can still use it if the binding object could be `null` or, as introduced in #94, a union of collection types. Therefore it may be more accurate to change the name of the `?.` operator to **“potential access”**.
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
`VoidError` was used to represent attempts at accessing a value that does not exist. This is renamed to `NullError`, and is used for attempting to access properties on `null`.
```cp
let tup: [int, float, ?: str] = [0, 1.1];
tup!.2.length; % NullError at runtime
```
