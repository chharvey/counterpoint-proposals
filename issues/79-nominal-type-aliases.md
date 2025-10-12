A type alias can be declared with `nominal` to create a **nominal type alias**:
```cp
type nominal Name = str;
type nominal Age  = int;
```
Nominal type aliases are expected to be assigned types of the *same name*, regardless of their definition.
To allow the assignment, we have to use a type claim (#82) to tell the type-checker that it’s intended.
```cp
let n: Name = "Alice";           %> TypeError: `str` not assinable to `Name`
let n: Name = "Alice" as <Name>; % no error
```
For example, given the type `type Person = [name: Name, age: Age];`, only values of type `Name` and `Age` can be assigned to its properties.
```cp
let p: Person = [
	name= "Alice",     %> TypeError: `str` not assinable to `Name`
	age=  42 as <Age>, % no error
];
```
Think of type `Name` as a narrowing of type `str` and type `Age` as a narrowing of type `int`. Since `Person#age` is of type `Age`, we need a type claim to sufficiently *narrow* the type.

We can always widen types. `Name` and `Age` are subtypes of `str` and `int` respectively, and of course, since `anything` is the Top Type, we can always assign any type (even if nominal) to it.
```cp
let n1: Name        = p.name; % ok
let a1: Age         = p.age;  % ok
let n2: str         = p.name; % ok (widening)
let a2: int         = p.age;  % ok (widening)
let var u: anything = p.name; % ok (widening)
set u               = p.age;  % allowed reassignment
```

Nominal typing rules supercede standard type theory rules. For example, even though `anything` is the Top Type and any type should be assignable to it, a nominal type defined as `anything` requires assigned expressions to have its type by name. This applies similarly to type unions. Even type `nothing`, the Bottom Type, is not assignable to a nominal type.
```cp
type nominal Top = anything;
claim unreachable: nothing;
let u: Top = unreachable;           %> TypeError: `nothing` not assignable to `Top`
let u: Top = "Alice";               %> TypeError: `str` not assignable to `Top`
let u: Top = "Alice" as <anything>; %> TypeError: `anything` not assignable to `Top`
let u: Top = unreachable as <Top>;  % ok
let u: Top = "Alice" as <Top>;      % ok

type nominal Primitive = null | bool | int | float | str;
let number: int | float = 42;
let p: Primitive = unreachable;                %> TypeError: `nothing` not assignable to `Primitive`
let p: Primitive = number;                     %> TypeError: `int | float` not assignable to `Primitive`
let p: Primitive = number as <int>;            %> TypeError: `int` not assignable to `Primitive`
let p: Primitive = number as <42>;             %> TypeError: `42` not assignable to `Primitive`
let p: Primitive = unreachable as <Primitive>; % ok
let p: Primitive = number      as <Primitive>; % ok
```

The motivation behind nominal types is that they’re useful for distinguishing different formats of data. For example, even though “string” (`str`) is one type, there can be different string formats: timestamps, UUIDs, numeric strings in different formats, and even source code such as JSON. The same goes for numbers, when dealing with currency or units (dimensional analysis) for example. Nominal typing requires us to be explicit when assigning primitive values and provides a double-check that, “yes, this is really what I meant to do”.

When *compound types* are declared `nominal`, we can also assign values with type claims.
```cp
type nominal Person = [name: str, age: int];
let p1: Person = [name= "Bob", age= 42];             %> TypeError: `[name: "Bob", age: 42]` not assignable to `Person`
let p2: Person = [name= "Bob", age= 42] as <Person>; % ok
```

When a `nominal` type alias is assigned a function type, only named functions that `impl` it (#84) may be assigned to it.
```cp
type nominal Operation = (float, float) => float;
function applyOperation(op: Operation): float => op.(3.0, 4.0);

applyOperation.(op= (a: float, b: float): float => a + b);                  %> TypeError
applyOperation.(op= ((a: float, b: float): float => a + b) as <Operation>); % ok, using type claim
applyOperation.(op= ((a, b) => a + b) as <Operation>);                      % more terse

function add(a: float, b: float): float => a + b;
function subtract(a, b) impl Operation => a - b;
applyOperation.(add);      %> TypeError
applyOperation.(subtract); % ok

let multiply: (a: float, b: float) => float = (a, b) => a * b;
let divide:   Operation                     = ((a, b) => a / b) as <Operation>;
applyOperation.(multiply); %> TypeError
applyOperation.(divide);   % ok
```
