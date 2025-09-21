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
