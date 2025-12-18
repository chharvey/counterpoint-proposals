**Generic function annotations** allow functions to implement generic types. Follows #84 and #85.

```cp
function lessThan1(a, b): int impl Comparator1 {
	; % ...
}
function lessThan2(a, b) impl Comparator2.<float> {
	; % ...
}
```

The only thing to remember here is that if the implemented signature takes a generic parameter, the generic parameter of the implmeneting function may be declared. This allows us to use the parameter as an argument of the annotation, if needed.
```cpl
typefunc Voider<A, B> => \(A, B) => void;

function myFunction<T>(x, y) impl Voider.<str, T> {
%                   ^ generic parameter is here, on the function name
	x; %: str
	y; %: T   % supplied at call sites
}

myFunction.<bool>("hello", false);
myFunction.<int>("hello", 42);
```
The function `myFunction` above has a generic parameter, which is required whenever the function is called. That parameter also happens to be used as an argument of the `Voider` generic type.

This next example uses a generic type function whose value is itself a generic function. Notice we may omit a function’s generic parameter if we’re not using it in the implementation call nor in the function body.
```cpl
typefunc Nuller<T> => \<U>(T, U) => null;

function g(i, j) impl Nuller.<int> {
%         ^ no generic parameter needed
	i; %: int
	j; %: U   % for some given U
	return null;
}

function h<U>(i, j) impl Nuller.<bool> {
%          ^ generic parameter needed if used in the body
	type V = U & null;
	i; %: bool
	j; %: U    % for some given U
	return <V>null;
}

g.(42, 4.2);            %> TypeError: Got 0 generic parameters, but expected 1.
g.<float>(42, 4.2);
h.<str>(true, "world");
```
A generic parameter is required at call sites of `g`, even though it was declared without any, because of the signature it implements.
