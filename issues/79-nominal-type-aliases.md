A type alias can be declared with `nominal` to create a **nominal type alias**:
```cp
type nominal Name = str;
type nominal Age  = int;
```
Nominal type aliases are expected to be assigned types of the *same name*, regardless of their definition. Thus, given the type
```cp
type Person = [name: Name, age: Age];
```
we can only assign its `name` and `age` properties values of type `Name` and `Age` respectively.
```cp
let p: Person = [
	name= "Alice", %> TypeError: `str` not assinable to `Name`
	age=  <Age>42, % no error
];
let n: Name = <Name>"Alice"; % no error
```
Think of type `Name` as a narrowing of type `str` and type `Age` as a narrowing of type `int`. Since `Person#age` is of type `Age`, we cannot assign an `int` without a type claim (#82).

We can always widen types. Both `Name` and `Age` are subtypes of `obj`, and of course, since `unknown` is the Top Type, we can always assign any type (even if nominal) to it.
```cp
let n1: Name    = p.name; % ok
let a1: Age     = p.age;  % ok
let n2: str     = p.name; % ok
let a2: int     = p.age;  % ok
let unfixed o: obj     = p.name; % ok
set o                  = p.age;  % ok
let unfixed u: unknown = p.name; % ok
set u                  = p.age;  % ok
```

The motivation behind nominal types are that they’re useful for distinguishing different formats of data. For example, even though “string” (`str`) is one type, there can be different string formats: timestamps, UUIDs, numeric strings in different formats, and even source code such as JSON. The same goes for numbers: when dealing with currency or units (dimensional analysis) for example. Nominal typing requires us to be explicit when assigning primitive values and provides a double-check that, “yes, this is really what I meant to do”.

When compound types are declared `nominal`, we can also assign values with type claims.
```cp
type nominal Person = [name: str, age: int];
let p1: Person = [name= "Bob", age= 42];         %> TypeError: `[name: "Bob", age: 42]` not assignable to `Person`
let p2: Person = <Person>[name= "Bob", age= 42]; % ok
```
