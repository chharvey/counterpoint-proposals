Value types and reference types are mutually exclusive in the type hierarchy, since they have completely different implementations. Value objects are usually pushed directly onto the operand stack, as opposed to being stored in the heap like reference objects are.

The `Object` class will be a class of which every reference object is an instance. There is no common ancestor class for value objects.

Built-in value objects are of type Null, Boolean, Number (Integer or Float), String, Tuple, or Record. Built-in reference objects are of type List, Dict, Set, or Map.

## Reserved Keywords
The `obj` keyword is retired and replaced with a new temporary keyword, `Object`, hard-coded into the grammar. Once we have the intrinsic `Object` class, it will be removed from the grammar and it will be considered an identifier instead (just like the plan for `List`, `Dict`, `Set`, and `Map`). `obj` can now be used as a regular identifier.
```cp
let o1: mut obj = my_object; % ReferenceError (`obj` was never declared)
let o2: obj = o1;            % ReferenceError (`obj` was never declared)

let o1: mut Object = my_object; % fixed
let o2: Object = o1;            % fixed

let obj: int = 42; % a regular identifier now

type obj = int; % a regular identifier now
```

## `Object` vs `unknown`
The `Object` type contains every instance of the `Object` class or any of its subclasses, which includes all reference objects. When a variable of type `Object` is assigned, it is only allocated enough space to hold a reference. We can assume a variable of type `Object` has all the fields and methods available via the `Object` class. Value types are not assignable to the `Object` type.

On the other hand, nothing is assumed about values of type `unknown`. The `unknown` type is the “Top Type” or “universal type”, and it also includes all value types and reference types. (Remember, `Object` only includes reference types). We cannot access properties on type `unknown`, since we can’t be guaranteed those properties even exist.

## Autoboxing
It’s not possible to store a value object where a reference is expected in the runtime environment, so we have to assign a reference in its place. When a value object is assigned to a variable/parameter of type `unknown`, the value is **autoboxed**.

In Java, the autoboxing mechanism used allows assigning a value object to a reference type, for example `Integer my_int_obj = 42;`. Java autoboxing is where the compiler internally creates an `Integer` object with a property of the value `42`, stores it in the heap, and assigns `my_int_obj` a reference to it, rather than the value itself. So, though it appears `my_int_obj` holds the value, it actually holds the reference to an ad hoc wrapper object.

Counterpoint takes a simpler approach, but dropping the ad hoc wrapper object serving as the middle-man. When assigning a value object to a reference type or type `unknown`, the value *itself*, unwrapped, is stored directly on the heap, and a reference is returned in its place. When referencing the `unknown`-type variable, we have direct access to its value rather than needing to “unwrap” it first.
```cp
% internally, these are all replaced with references
let unn1: unknown = null;
let unn2: unknown = false;
let unn3: unknown = 42;
let unn4: unknown = 4.2;
let unn5: unknown = "hello";

unn5 === "hello"; % still value-identical

% internally, a list of references, not value objects
List.<unknown>([
	null,
	true,
	421,
	4.21,
	"world",
]);

class data FullNameVal {
	new (
		public const first: str,
		public const last:  str,
	) {;}
	public fullName(): str
		=> """{{ this.first }} {{ this.last }}""";
}
let full_name_ref: unknown = FullNameVal.("Isaac", "Newton"); % autoboxed

full_name_ref === FullNameVal.("Isaac", "Newton"); % still value-identical
full_name_ref.fullName.() == "Isaac Newton";       % still has access to methods
```

Autoboxing only applies to assignees of type `unknown`. Otherwise, assignment of a value type to a reference type is not allowed.
```cp
interface FullNameRef {
	readonly first: str;
	readonly last:  str;
}
let full_name_ref1: FullNameRef = FullNameVal.("Isaac", "Newton"); %> TypeError
let full_name_ref2: Object      = FullNameVal.("Isaac", "Newton"); %> TypeError
```

Autoboxing can be perfomed at the parameter level and at the expression level as well.
```cp
function take_unknown(u: unknown): void {;}

take_unknown.(FullNameVal.("Isaac", "Newton")); % autoboxing: assigning a value object to a reference-type parameter

<unknown>FullNameVal.("Isaac", "Newton"); % autoboxing: type-claim to `unknown`
```

Autoboxing is less performant than working directly on the operand stack, but it allows dynamic typing for variables and parameters. Of course, now that the value is stored on the heap instead of the stack, some memory cleanup will have to take place once the value is no longer used.

## Unboxing
By default, it is still a type error to assign a reference object to a value type (even if it’s a correct subtype).
```cp
let ref0: unknown = 42;   % autoboxed
let val0: int     = ref0; %> TypeError

let full_name_val: FullNameVal = full_name_ref; %> TypeError
```
This error was introduced in #90 because we didn’t want a reference object mutating, which would then be observed in the value object. However, we are introducing a new mechanism called **unboxing**, which adds some leniency. When an autoboxed value is assigned back to a value type and *explicitly unboxed*, the value is *copied* directly into the variable. The reference to the original object is still kept, but any changes to it aren’t observed in the new variable.

To explicilty unbox an autoboxed variable, simply use the **type-claim** operator.
```cp
let ref0: unknown = 42;        % autoboxed
let val0: int     = <int>ref0; % unboxing is performed: a copy of ref0’s value is assigned to val0

val0 === 42;   % still value-identical
val0 === ref0; % still value-identical

let full_name_ref: unknown     = FullNameVal.("Isaac", "Newton"); % autoboxed
let full_name_val: FullNameVal = <FullNameVal>full_name_ref;      % unboxed and copied

full_name_ref === FullNameVal.("Isaac", "Newton"); % still value-identical
full_name_val === full_name_ref;                   % still value-identical
full_name_val.fullName.() == "Isaac Newton";       % still has access to methods
```
Unboxing may only be performed on previously autoboxed *value objects*. Reference objects cannot be unboxed. Because of this, autoboxed values cannot be mutated.

```cp
class Point {
	new (
		public const x: str,
		public const y: str,
	) {;}
}
interface data PointLike {
	readonly x: str;
	readonly y: str;
}
let point:     unknown   = Point.(3, 5);     % not autoboxed; the new object is already in the heap
let pointlike: PointLike = point;            %> TypeError
let pointlike: PointLike = <PointLike>point; %> TypeError

let full_name_val:  FullNameVal = FullNameVal.("Isaac", "Newton"); % original value object
let full_name_ref:  unknown     = full_name_val;                   % autoboxed
let full_name_val2: FullNameVal = <FullNameVal>full_name_ref;      % unboxed and copied

set full_name_val.first = "Albert"; %> MutabilityError
set full_name_ref.first = "Albert"; %> TypeError
```
