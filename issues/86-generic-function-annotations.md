**Generic function annotations** allow functions to implement generic types. Follows #84 and #85.

```cp
function lessThan1<U> implements Comparator1 (a: U, b: U): int {
	; % ...
}
function lessThan2 implements Comparator2.<float> (a: float, b: float): int {
	; % ...
}
```

The only thing to remember here is that the generic parameters of a function declaration come right after the function name, *before* the `implements` clause. This allows us to use the parameter as an argument of the annotation, if needed.
```cp
type Voider<A, B> = (A, B) => void;

function myFunction<T> implements Voider.<str, T> (x: str, y: T): void {
%                   ^ generic parameter is here, on the function name
	;
}

myFunction.<bool>("hello", false);
myFunction.<int>("hello", 42);
```
The function `myFunction` above has a generic parameter, which is required whenever the function is called. That parameter also happens to be used as an argument of the `Voider` generic type.

More examples:
```cp
type Nuller<T> = <U>(T, U) => null;

function g<U> implements Nuller.<int> (i: int, j: U): null {
	i; j; return null;
}
function h<U> implements Nuller.<bool> (i: bool, j: U): null {
	i; j; return null;
}

g.<float>(42, 4.2);
h.<str>(true, "world");
```

And as mentioned in #84, functions that have an `implements` clause don’t need to have explicit parameter/return types written.
```cp
function myFunction<T> implements Voider.<str, T> (x, y) {
	x; % implied `str`
	y; % implied `T` (specified when function is called)
}

function g<U> implements Nuller.<int> (i, j) {
	i; %: int
	j; %: U
	return null;
}
function h<U> implements Nuller.<bool> (i, j) {
	i; %: bool
	j; %: U
	return null;
}
```
