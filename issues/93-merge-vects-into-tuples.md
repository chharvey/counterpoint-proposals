This issue supercedes #51, #56, and #90. It merges the functionality/semantics of Vects and Structs into that of Tuples and Records, respectively, while removing the “constant collection” (Vect and Struct) syntax. It also makes the equivalent change on the “type” side of things.

## Value Types are Immutable
Tuple types and Record types are now value types. Value types are always immutable, meaning that their entries cannot be reassigned. They keep their same syntax, while the syntax of Vects and Structs (`\[...]`) is removed.
```cp
let foo: [bool, int, str] = [false, 42, "hello"];
set foo.0 = true; %> MutabilityError: Mutation of an object of immutable type `[bool, int, str]`.

let bar: mutable (bool | float)[2] = [4.2, true]; %> TypeError: Invalid operation.

type Diz = \[bool, int]; %> ParseError
let diz: Diz = \[false, 43]; %> ParseError
```

## Value Types are Identical
As discussed in #51, value objects that have the “same value” are identical, and that recursively includes their entries. This contrasts with reference objects, which are only identical when they are the same reference.
```cp
let val1: [bool, int, str] = [false, 42, "hello"];
let val2: [bool, int, str] = [false, 42, "hello"];
val1 === val2; %== true

let ref1: List.<bool> = List.<bool>([false, true]);
let ref2: List.<bool> = List.<bool>([false, true]);
ref1 ==  ref2; %== true
ref1 !== ref2; %== true
ref1 === ref1; %== true
```

## Copied by Value
Value objects are copied by value whenever assigned to variables/parameters of the same type. (If the variable/parameter is a reference type, the value is autoboxed.) Assigning a variable to another variable results in a copy of the value type being made and assigned to that variable, rather than a reference pointing to the same object.
```cp
let foo: [bool, int, str] = [false, 42, "hello"];
let bar: [bool, int, str] = foo;
foo.0 === bar.0; %== true
foo.1 === bar.1; %== true
foo.2 === bar.2; %== true
```
(Since value objects are immutable, there’s no observable difference had they been reference objects.)

The List, Set, and Map constructors may still take Tuples as arguments, even though Tuple is now a value type. And the Dict constructor may still take Records as arguments, even though Record is now a value type.
```cp
List.<int>([1, 2, 3]);
Dict.<int>([a= 1, b= 2, c= 3]);
Set.<int>([1, 2, 3]);
Map.<int, float>([
	[1, 0.1],
	[2, 0.2],
]);
```

## Compound Value Types
**New:** Value objects may now contain pointers. For example, a Tuple may contain a reference in one of its entries. That reference may even point to a mutable object!
```cp
type T = [
	ints:   List.<int>,
	floats: mutable List.<float>,
]; % no longer a Type Error!

let qux: T = [
	ints=   List.<int>(),
	floats= List.<float>(),
]; % no longer a Type Error!

set qux.floats.[0] = 42; % mutating an object referenced in a value type
```
When value objects are compared by identity, any references they contain are also compared by identity. This means that two different references, even if they have the same type, will make the value objects non-identical.
```cp
let qux: [ints: List.<int>] = [ints= List.<int>()];
let diz: [ints: List.<int>] = [ints= List.<int>()];
qux !== diz; %== true
```
Because compound value types may contain references, it’s possible for value objects to be equal but not identical. This was not previously the case — all value objects that were equal were necessarily identical — but the example above illustrates how that’s no longer true. Even though `qux !== diz`, they are equal by composition (because `qux.ints == diz.ints`), therefore `qux == diz`.

This is nothing new for value objects. The floating values `0.0` and `-0.0` are equal (`0.0 == -0.0`) but not identical (`0.0 !== -0.0`).

## Assignability
When a value object is assigned to a reference type, it is autoboxed;
when a reference object is assigned to a value type, it is unboxed.
As always, value objects — or any immutable object for that matter — cannot be assigned to a mutable type.
```cp
% still allowed: assigning value objects to non-mutable reference types
% (autoboxing is performed)
let a: Object = [42, 420, 4200];
let b: Object = [n42= 42, n420= 420];
let c: interface { n42: int; n420: int; } = [n42= 42, n420= 420];

% still errors: cannot assign value objects to mutable reference types
let d: mutable Object = [42, 420, 4200];                                    %> TypeError
let e: mutable Object = [n42= 42, n420= 420];                               %> TypeError
let f: mutable (interface { n42: int; n420: int; }) = [n42= 42, n420= 420]; %> TypeError

% still allowed: assigning reference objects to value types
% (unboxing is performed)
let h: int[3] = <int[3]>a;
```
