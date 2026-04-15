A type alias can be declared with `nominal` to create a **nominal type alias**:
```cpl
type nominal Name = str;
type nominal Age  = int;
```
Nominal type aliases are expected to be assigned types of the *same name*, regardless of their definition.
To allow the assignment, we have to use a type claim (#82) to tell the type-checker that it’s intended.
```cpl
val n: Name = "Alice";           %> TypeError: `str` not assinable to `Name`
val n: Name = "Alice" as :Name:; % no error
```
For example, given the record type `type Person = (name: Name, age: Age);`, only values of type `Name` and `Age` can be assigned to its properties.
```cpl
val p: Person = (
	name= "Alice",     %> TypeError: `str` not assinable to `Name`
	age=  42 as :Age:, % no error
);
```

The motivation behind nominal types is that they’re useful for distinguishing different formats of data. For example, even though “string” (`str`) is one type, there can be different string formats: timestamps, UUIDs, numeric strings in different formats, and even source code such as JSON. The same goes for numbers, when dealing with currency or units (dimensional analysis) for example. Nominal typing requires us to be explicit when assigning primitive values and provides a double-check that, “yes, this is really what I meant to do”.

A nominal type behaves similarly to the Bottom Type in that nothing except itself (and the Bottom Type) is assignable to it. Thinking about nominal types this way lets us reason about relationships and assignability. Type `Name` is like a narrowing of type `str`, and type `Age` is like a narrowing of type `int`. We need type claims to sufficiently narrow a value’s type to let it be assigned.

We can always widen types. `Name` and `Age` are subtypes of `str` and `int` respectively, and of course, since `anything` is the Top Type, we can always assign any type (even if nominal) to it.
```cpl
val n1: Name        = p.name; % ok
val a1: Age         = p.age;  % ok
val n2: str         = p.name; % ok (widening)
val a2: int         = p.age;  % ok (widening)
val mut u: anything = p.name; % ok (widening)
set u               = p.age;  % allowed reassignment
```
Of course, nominal typing rules do not override standard hierarchical type theory rules. The actual Bottom Type `nothing` is assignable to any and all types, even nominal types.
```cpl
claim unreachable: nothing;
val n3: Name = unreachable; % ok (widening)
val a3: Age  = unreachable; % ok (widening)
```

Nominal types are separate from each other in the type hierarchy. Even though both `Name` and `Age` behave like the Bottom Type, it’s more accurate to think of them as *their own* bottom types. We can’t even assign two nominal types to each other when they have the same definition!
```cpl
type nominal Name     = str;
type nominal EmplId   = str;
type nominal Position = str;
interface User {
	name:         Name;
	id:           EmplId;
	mut position: Position;
};
claim u1: User;
claim u2: User;

set u1.position = u2.name;                             %> TypeError: Name is not assignable to Position
set u1.position = u2.name as :Position:;               %> TypeError: Name and Position have no overlap
set u1.position = u2.name as :anything: as :Position:; % allowed, but not recommended
```
Even though `nominal Name` and `nominal Position` are defined as strings, we cannot assign them to one another without type-claiming. And since they have no overlap, a direct type claim will still raise an error; we’d need to widen to `anything` before narrowing again. The double claim is required to prevent accidental footguns.

Nominal types may be unions as well; they follow all the same rules.
```cpl
type nominal Primitive = null | bool | sym | int | nat | float | str;
val number: int | nat | float = 42;
val p: Primitive = number;                %> TypeError: `int | float` not assignable to `Primitive`
val p: Primitive = number as :int:;       %> TypeError: `int` not assignable to `Primitive`
val p: Primitive = number as :42:;        %> TypeError: `42` not assignable to `Primitive`
val p: Primitive = number as :Primitive:; % ok
```

When *compound types* are declared `nominal`, we can also assign values with type claims.
```cpl
type nominal Person = (name: str, age: int);
val p1: Person = (name= "Bob", age= 42);             %> TypeError: `(name: "Bob", age: 42)` not assignable to `Person`
val p2: Person = (name= "Bob", age= 42) as :Person:; % ok
```

When a `nominal` type alias is assigned a function type, only named functions that `impl` it (#84) may be assigned to it.
```cpl
type nominal Operation = \(float, float) => float;
func applyOperation(op: Operation): float => op.(3.0, 4.0);

applyOperation(op= \(a: float, b: float): float => a + b);                  %> TypeError
applyOperation(op= (\(a: float, b: float): float => a + b) as :Operation:); % ok, using type claim
applyOperation(op= (\(a, b) => a + b) as :Operation:);                      % more terse

func add(a: float, b: float): float => a + b;
func subtract(a, b) impl Operation => a - b;
applyOperation(add);      %> TypeError
applyOperation(subtract); % ok

val multiply: \(a: float, b: float) => float = \(a, b) => a * b;
val divide:   Operation                      = (\(a, b) => a / b) as :Operation:;
applyOperation(multiply); %> TypeError
applyOperation(divide);   % ok
```
