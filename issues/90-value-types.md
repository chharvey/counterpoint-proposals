Value types are copied by value and are identical if they have the same value. Reference types are copied by refrence and are identical only by reference. This issue implements the distinction and relaxes some syntax restrictions.

# Discussion
Constant collections (#51) were syntax productions only a few additional semantics, which were applied on an ad-hoc basis. This issue strengthens those semantics by explicitly tagging objects as either reference objects or value objects, and types as either reference types or value types. This improvement takes a widespared approach, rather than specifically targeting on a case-by-case basis.

## Syntax Updates
The first change is a relaxation on syntax and an improvement on semantics. #51 only allowed a certain subset of syntaxes within the delimiters `\[` and `]`. Now, `\[…]` represents *value* types/objects and `[…]` represents *reference* types/objects. These syntaxes may now contain identifiers and function calls, as long as those return value types/objects, determined by semantic analysis. Under the hood, a tag will be implemented on each type to indicate whether it’s a value or reference type.
```cp
% constant collections now may include variable references and function calls
let x: Object = \[a, b.(c.d), e.f.[g]];
type X = \[A, B.<C.D>, E.F[3]];
```
Note that these expressions no longer have to be foldable! Value objects may exist at runtime, as long as they only contain other value objects. In the example above, `a` might be an unfixed variable and `b.(c.d)` might only be computable at runtime; both of these expressions are dynamic (unknown to the compiler). However, semantic analysis at compile-time will allow or disallow the entries based on their type signature.
```cp
let x: Object = \[a];       % TypeError only if `a` is a reference object
let y: Object = \[b.()];    % TypeError only if `b` has a reference type return signature
let z: Object = \[b.(c)];   % not a TypeError if `b` returns a value type, even if `c` is a reference object!

type X = \[A];     % TypeError only if `A` is a reference type
type Y = \[B.<C>]; % not a TypeError if B.<C> is a value type, even if `C` is a reference type!
```
(As an example of `B.<C>` being a value type that accepts a reference type generic: say we define `type B.<C> = (C) => \[int]` — a function type that takes an argument of any type and returns the value type `\[int]`.)

## Assignability Updates
Another change affects assignability — Only value objects can be assigned to value types. Value objects can still be assigned to reference types, just so long as they’re not mutable.

TLDR:
- A value object is always assignable to a value type (assuming it’s a proper subtype).
- A reference object is always assignable to a refrence type (assuming it’s a proper subtype, or, if the assigned expression is a collection literal, its entries correspond properly).
- A value object is *sometimes* assignable to a reference type (assuming it’s a proper subtype), as long as the assignee type is not mutable.
- **(NEW):** A reference object is *never* assignable to a value type, even if the assignee type is non-mutable.

```cp
% still allowed: assigning value objects to value types
let c: int\[3] = \[42, 420, 4200];
let d: \[n42: int, n420: int] = \[
	n42=  42,
	n420= 420,
];

% still allowed: assigning value objects to non-mutable reference types
let a: int[3] = \[42, 420, 4200];
let b: [n42: int, n420: int] = \[
	n42=  42,
	n420= 420,
];

% still errors: cannot assign value objects to mutable reference types
let e: mut int[3] = \[42, 420, 4200];
let f: mut [n42: int, n420: int] = \[
	n42=  42,
	n420= 420,
];

% NEW ERRORS! cannot assign reference objects to value types
let g: int\[3] = [42, 420, 4200];
let h: \[n42: int, n420: int] = [
	n42=  42,
	n420= 420,
];
```
The types `int\[3]` and `\[n42: int, n420: int]` are **value types** and can only be assigned value objects.


## Type Operation Updates
Value types can no longer have the `mut` operator applied.
```cp
% invalid:
type T = mut \[int, float, str];
type T = mut never;
type T = mut void;
type T = mut null;
type T = mut bool;
type T = mut int;
type T = mut float;
type T = mut str;

% valid:
type T = mut Object;  % still valid, but just returns `Object`
type T = mut unknown;
type T = mut (\[int, float] | [int, float]); % still valid even though result won’t be mutable
type T = \[int, float] | mut [int, float];
```
