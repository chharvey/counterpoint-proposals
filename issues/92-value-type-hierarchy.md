Value types and reference types are mutually exclusive in the type hierarchy, since they have completely different implementations. Value objects are usually pushed directly onto the operand stack, as opposed to being stored in the heap like reference objects are.

The `Object` class will be a class of which every reference object is an instance. There is no common ancestor class for value objects, but they are all assignable to `Object` (more on that below).

## Reserved Keywords
The `obj` keyword is retired and replaced with a new temporary keyword, `Object`, hard-coded into the grammar. Once we have the `Object` class, it will be removed from the grammar and it will be considered an identifier instead (just like the plan for `List`, `Dict`, `Set`, and `Map`). `obj` can now be used as a regular identifier.
```cp
let o1: mut obj = my_object; % ReferenceError (`obj` was never declared)
let o2: obj = o1;            % ReferenceError (`obj` was never declared)

let o1: mut Object = my_object; % fixed
let o2: Object = o1;            % fixed

let obj: int = 42; % a regular identifier now
```

## `Object` vs `unknown`
The `Object` type contains every instance of the `Object` class, which includes all reference objects. When a variable of type `Object` is assigned, it is only allocated enough space to hold a reference. We can assume a variable of type `Object` has all the fields and methods available to them via the `Object` class.

On the other hand, nothing is assumed about values of type `unknown`. The `unknown` type is the “Top Type” or “universal type”, and it also includes type `void`, which `Object` does not. We cannot access properties on type `unknown`, since we can’t be guaranteed those properties even exist.

Even though value objects are not technically instances of `Object`, they can still be assigned to variables/parameters of type `Object` as discussed below. They can also be assigned to type `unknown`.

## Autoboxing
It’s not possible to store a value object where a reference is expected in the runtime environment, so we have to assign a reference in its place.

When a value object is assigned to a variable/parameter of type `Object` or `unknown`, the value is **autoboxed**. And in general, autoboxing is done any time a value object is assigned to a reference type (assuming it passes type-checking).

In Java, the autoboxing mechanism used allows assigning a value object to a reference type, for example `Integer my_int_obj = 42;`. Java autoboxing is where the compiler internally creates an `Integer` object with a property of the value `42`, stores it on the heap, and assigns `my_int_obj` a reference to it, rather than the value itself. So, though it appears `my_int_obj` holds the value, it actually holds the reference to an ad-hoc wrapper object.

Counterpoint takes a simpler approach, but dropping the ad-hoc wrapper object serving as the middle-man. When assigning a value object to a reference type or type `unknown`, the value *itself*, unwrapped, is stored directly on the heap, and a reference is returned in its place. When referencing the variable, we have direct access to the value rather than needing to “unwrap” it first.
```cp
% internally, these are all replaced with references
let unn1: unknown = null;
let unn2: unknown = false;
let unn3: unknown = 42;
let unn4: unknown = 4.2;
let unn5: unknown = "hello";

% internally, a list of references, not value objects
List.<Object>([
	null,
	true,
	421,
	4.21,
	"world",
]);

unn5 == "hello"; %== true
```
Autoboxing is less performant than working directly on the operand stack, but it allows dynamic typing for variables and parameters. Of course, now that the value is stored on the heap instead of the stack, some memory cleanup will have to take place once the value is no longer used.

In general, whenever a value object is assigned to a reference type, it is autoboxed.
```cp
% autoboxing is performed
let a: int[3] = \[42, 420, 4200];
let b: [n42: int, n420: int] = \[n42= 42, n420= 420];

% FUTURE:
let c: interface { first: str; last: str; } = (class data {
	new (public first: str, public last: str) {;}
}).("Isaac", "Newton");
```

## Unboxing
It’s no longer a type error to assign a reference object to a value type (as long as it’s a correct subtype). This error was introduced in #90 because we didn’t want that reference object mutating, which would then be observed in the value object. However, we are introducing a new mechanism called **unboxing**, which eliminates the problem. When the assignment takes place, its value is *unboxed and copied* directly into the variable. The reference to the original object is still kept, but any changes to it aren’t observed in the new variable.
```cp
let ref0: Object = 42;
let val0: int = <int>ref0; % unboxing is performed: a copy of ref0’s value is assigned to val0
ref0 === 42;
val0 === 42;

let ref1: mut int[3] = [42, 420, 4200];
let val1: int\[3] = ref1;    % unboxing
set ref1.0 = 42000;          % ref1 is mutated
ref1 == \[42000, 420, 4200]; % observe ref1 change
val1 === \[42, 420, 4200];   % val1 is unchanged

let ref2: mut [n42: int, n420: int] = [n42= 42, n420= 420];
let val2: \[n42: int, n420: int] = ref2; % unboxing
set ref2.n42 = 42000;
ref2 == \[n42= 42000, n420= 420];
val2 === \[n42= 42, n420= 420];

% FUTURE:
class FullName {
	new (public first: str, public last: str) {;}
}
interface data FullNameData {
	readonly first: str;
	readonly last:  str;
}
let ref3: mut FullName = FullName.("Isaac", "Newton");
let val3: FullNameData = ref3; % unboxing
set [ref3.first, ref3.last] = ["Albert", "Einstein"];
ref3 == FullName.("Albert", "Einstein");
val3 == FullName.("Isaac", "Newton");
```
