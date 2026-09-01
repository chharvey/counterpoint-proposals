# Structural Typing
Counterpoint is **nominally typed** by default, which means types are compared by name.

Nominal typing is based on this fundamental rule:
> **A value can only be assigned to a type `T` if it was constructed with the `T` constructor (if `T` is a class),**
> **or the constructor of some class that explicitly implements `T` (if `T` is an interface).**

For example, only a Point object can be assigned to a variable of type Point:
```cpl
class Point {
	public new (
		public readonly x: float,
		public readonly y: float,
	) {}
}
val p: Point = (x= 2.4, y= 4.2); %> TypeError
val p: Point = Point(2.4, 4.2);  % ok
```
Even though the record has all the same properties and types of `Point`,
it wasn’t constructed with a class of the same *name*, so the assignment is not allowed.

Nominal typing can be overridden with structural typing where desired.
For example, if all we cared about was that `p` had properties `x` and `y`,
each of type `float`, then we want to be able to let the record assignment pass.

In that case, we can use the `struct` type operator on the variable assignee type.
```cpl
val p: struct Point = (x= 2.4, y= 4.2); % ok
```
The type `struct Point` allows **structural typing** to take place,
where only the “shape” (the type of its public members) is considered during assignment,
rather than the names and positions of types in the type hierarchy.

In general, a type `B` is considered a structural subtype of `A` if every member of `A` appears in `B` and
each of these members in `B` is compatible with its corresponding member in `A`.
This is known as the **Liskov Substitution Principle**:
if the source type is acceptable everywhere the target type is expected,
then the source is a sufficient subtype of the target.

The rules are a little more complicated than introduced above,
but are fully discussed in the [Variance](./variance.md) chapter.
But as an outline,

In structural typing, a type `B` is considered a subtype of `A` if:
- every required member of `A` is also required in `B`, and
- the type of each member in `B` meets at least one of the following conditions:
	- there is no corresponding member of `A`
	- is unrelated to the corresponding member of `A`
		— as long as `A` is bivariant with that type, or that member in `A` is write-only and `A` is not mutable
	- is a subtype of the corresponding member of `A`
		— as long as `A` is covariant with that type, or that member in `A` is read-only (or it’s read-write and `A` is not mutable), or,
		the members are methods
	- is a supertype of the corresponding member of `A`
		— as long as `A` is contravariant with that type, or that member in `A` is write-only
	- is equal to the corresponding members of `A`
		— as long as `A` is invariant with that type, or that member in `A` is read-write

Structural typing is recursive: the compiler must evaluate a type’s properties
in order to evaluate the type itself. This could lead to infinite recursion.
```cpl
class Person {
	public name: str;
	public friend: Person;
}
class Robot {
	public name: "Foo" | "Bar";
	public friend: Robot;
	public serial: int;
}
val p: struct Person = Robot();
```
When the compiler reaches the last line above, it needs to determine whether a `Robot` is structurally assignable to `Person`,
which would involve comparing the properties of each type.
Specifically, it needs to check the two fields of the target type: `name` and `friend`.
`"Foo" | "Bar"` is assignable to `str`, so the `name` field passes.
But now it has to check whether the type of `Robot#friend` is a subtype of the type of `Person#friend`,
but because `Robot#friend` is `Robot`, and `Person#friend` is `Person`,
it ends up needing to check whether `Robot` is a subtype of `Person`!

To resolve this infinite recursion, Counterpoint takes an “innocent until proven guilty” approach:
First before determining fully whether `Robot` is a subtype of `Person`, we assume it is true,
and then we use that assumption in any sub-calculations we need to make.
In the case above, when checking whether `Robot#friend` is a subtype of `Person#friend`,
we the assumption that `Robot` is a subtype of `Person`, which passes the field.
Since all the other fields pass, then the calculation returns true.
To sum it up, **indistinguishable types should be considered equal.**

Structural typing is not the only strategy that Counterpoint uses when determining assignability.
If `Robot` is assignable to `Person`, we may want to declare this relationship explicitly by
making it a [subclass](./inheritance.md).
Then the compiler only has to check assignability once, in the `extends` clause, rather than on each assignment.
```cpl
class Person {
	public name: str;
	public friend: Person;
}
class Robot extends Person { % assignability is determined on this line …
	public claim name: "Foo" | "Bar";
	public claim friend: Robot;
	public serial: int;
}
val p: struct Person = Robot(); % … rather than on this line.
```
Because `Robot extends Person`, we now have a nominal, hierarchical, relationship:
`Robot` is a subtype of `Person` by its definition, so we don’t even need to check its members to determine assignability.
Even though the target type is `struct Person`, the compiler still uses nominal typing to allow the assignment.

To think about this another way: `Person` is a subtype of `struct Person` (and in fact this is true for any type).
So if `Robot extends Person`, then by transitivity, `Robot` is certainly a subtype of `struct Person`.



## Runtime Operations
The equality operator `==` and the “instance-of” operator `is` always use nominality, even when one of the operands has a structural type.
Structural typing and assignability happens at copmile-time, whereas these relations are executed at runtime.
**Nominal types are considered** when checking two values for equality and when checking instances,
even if one or both values has a structural type.
```cpl
class Point {
	public new (
		public readonly x: float,
		public readonly y: float,
	) {}
}
val p1: Point = Point(2.4, 4.2);
val p2: struct Point = (x= 2.4, y= 4.2);
p1 == p2; %== false

func is_actual_point(p: Point): true
	=> p is Point; % definitely true

func is_point_like(p: struct Point): true
	=> p is Point; %> TypeError: Expression of type `bool` is not assignable to type `true`.
```
Even though the equality algorithm checks for deep-equality, and it is true that `p1.x == p2.x && p1.y == p2.y`,
they are still not equal because they were constructed with different constructors:
`p1` was constructed with `Point` and `p2` was constructed as a record literal.
This construction information is preserved at runtime, whereas the type declaration is lost by then.

`p is Point` returns true iff `p` is an instance of the `Point` class at runtime;
if `p` has a nominal type `Point`, the type-checker guarantees it will be true.
If however `p` only has a structural type `Point`, then we’re only guaranteed that `p`
satisfies the `Point` type at compile-time, but we don’t know for sure whether `p`
is an actual instance of the `Point` class.

For any type `T`, `struct T` is always a strict supertype of `T`.
A variable of type `struct Point` could be constructed with the `Point` constructor,
or it could be constructed with a record, or some other unrelated class,
so we can’t assign it back to the nominal `Point` type.
```cpl
claim p2: struct Point;
claim print_point(p: Point): void;
print_point(p2); %> TypeError
```



## Structural Interfaces
If an interface `extends` a class, then every class that implements that interface must *explicitly* extend the parent class.
```cpl
class Animal {
	public eat(): void {}
	public move(): void {}
}
interface Swimmer extends Animal {
	isWet: bool;
	swim(): void;
}
class FakeDolphin impl Swimmer { % InheritanceError
	public eat(): void {}
	public move(): void {}
	public isWet: bool = false;
	public swim(): void {}
}
class RealDolphin extends Animal impl Swimmer { % ok
	public isWet: bool = false;
	public swim(): void {}
}
```
> InheritanceError: Class `FakeDolphin` implements interface `Swimmer` but does not extend class `Animal`.

Since `Swimmer` extends `Animal`, any class that implements `Swimmer` must explicitly extend `Animal` (or a subclass thereof).
Even though `FakeDolphin` implements `Swimmer` and is structurally compatible with `Animal`, we get an InheritanceError.

However, if we don’t care whether `FakeDolphin` is actually a *subclass* of `Animal`,
we only care that it shares its shape, then we have two options:
1. either remove the `impl Swimmer` clause from `FakeDolphin` and implement everything manually, or
2. update the heritage of `Swimmer` from `extends Animal` to `inherits Animal`.

We should only use the first option if we still want all `Swimmer` instances to be `Animal` instances,
or if we don’t have permissions to update `Swimmer`. However, there are risks to this option,
such as `FakeDolphin` falling out of sync with `Animal`.

With the second option, we can make it so that `Swimmer` only inherits the *members* of `Animal`, without being a nominal subtype of it:
```cpl
interface Swimmer inherits Animal {
%                 ^ changed from `extends`
	isWet: bool;
	swim(): void;
}
class FakeDolphin impl Swimmer { % no error
	public eat(): void {}
	public move(): void {}
	public isWet: bool = false;
	public swim(): void {}
}
```
The `inherits Animal` clause essentially treats `Animal` as if it were an interface instead of a class.
`FakeDolphin` can implement `Swimmer` (and by extension, `Animal`) without needing to be an explicit subclass of `Animal`.
And it won’t fall out of sync: if `Animal` were to add a new method, then `FakeDolphin` would need to do so as well.
Note: because we’ve now relaxed the the definition of `Swimmer`, this means *any* of its subclasses may play by the same rules:
they also don’t need to extend `Animal`.



## Class Constructor Shorthands
Each of the primitive literal types as well as four compound types have a literal constructor syntax:

Type        | Type Shorthand Syntax | Literal Constructor Syntax (example)
----------- | --------------------- | --------------------------
`Null`      | `null`                | `null`
`Boolean`   | `bool`                | `false`, `true`
`Symbol`    | `sym`                 | `@hello`
`Integer`   | `int`                 | `2`
`Natural`   | `nat`                 | `+4`
`Float`     | `float`               | `2.4`
`String`    | `str`                 | `"hello"`
`List[T]`   | `[T]`                 | `[1, 2, 3]`
`Dict[T]`   | `[:T]`                | `[a= 1, b= 2, c= 3]`
`Set[T]`    | `{T}`                 | `{1, 2, 3}`
`Map[T, U]` | `{T -> U}`            | `{"a" -> 1, "b" -> 2, "c" -> 3}`

(There are also syntaxes for tuple and record literals, but they have no corresponding class constructor.)
