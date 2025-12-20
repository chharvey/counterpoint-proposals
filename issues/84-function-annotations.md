Function annotations specify a contract for functions’ type signatures.

# Discussion

A declared function may be specified to implement a type signature using the `impl` keyword. This provides a contract for a function to adhere to, for more robustness.

## Motivation
A function’s type signature is basically its *type*, with additional information such as parameter names and optionality. Since function declarations have types in their syntax, their function signatures are built in.

```cpl
func add(x: float, y: float): float => x + y;
%         ^         ^       ^ these types form the type signature
add; %: (x: float, y: float) => float
```

Type signatures are useful in higher-order functions. For example, the type signature of a typical list folding (or “reducing”) function would include a function type as a parameter. We can use our `add` function above as a reducer of a list of floats.
```cp
claim foldList: <T>(list: List.<T>, reducer: (T, T) => T) => T;
let sum: float = foldList.<float>([4.2, 40.2], add); %== 44.4
```
In fact, `add` could be used many times in dozens of higher-order functions like `foldList`. But what if the type signature of `add` changes? This could break our function call.
```cpl
func add(x: float, y: float, z: float): float => x + y + z;
let sum: float = foldList.<float>([4.2, 40.2], add); %> TypeError
```
> TypeError: `(x: float, y: float, z: float) => float` not assignable to `(float, float) => float`.

It’s a good thing this error is raised, because it’s warning us that the function type is not valid where it’s used. But the problem is, the error is reported at the *call sites*, wherever the function is used, instead of just once at the *source site*. Making one change to `add` could result in dozens of new compile-time errors, possibly far away from where the change even occurred. It would be nice to have a stricter form of function signature, to warn us that we shouldn’t change `add` in the first place.

## Description
This is where function annotations save the day. Functions can be **annotated** with the `impl` keyword, indicating their type signature must match the given type.
```cpl
type BinaryOperatorFloat = (float, float) => float;
func add(x, y) impl BinaryOperatorFloat {
	x; %: float
	y; %: float
	return x + y; %: float
}
```
Since the type signature of `foldList.<float>` expects a `(float, float) => float`, providing an implementation of `BinaryOperatorFloat` suffices.
```cp
let sum: float = foldList.<float>([4.2, 40.2], add); % ok
```
When annotating a function type implementation, explicit parameter and return type annotations must be removed. The type-checker uses the `impl` clause to determine the function type, so explicit annotations are redundant.
```cpl
func add(x: float, y: float): float impl BinaryOperatorFloat => x + y; %> SyntaxError
```

Now when we make a breaking change to `add`, we get only one new error, right at the source of that change.
```cpl
func add(x, y, z) impl BinaryOperatorFloat => x + y + z; %> TypeError
let sum: float = foldList.<float>([4.2, 40.2], add); % error is not reported here
```
> TypeError: Got 3 parameters, but expected 2.
