Generic functions are functions whose type signatures depend on variable types.

# Discussion
A generic function is a function with one or more generic parameters. When the function is called, it must be given types as generic arguments. Generic functions are useful when its parameter types or return type depends on a given type.
```cp
func iden<T>(v: T): T {
	return v;
}
```
The function above has a generic parameter `T`, takes an argument of that type, and returns a value of that type. When the function is called, the generic is specified.
```cp
let x: int = iden.<int>(42);
```

Generic parameters are different from the top type, `unknown`, and allow for finer grained control of the type sysstem. In the example above, we can declare `x` as type `int` because `iden` is generic. If it weren’t, then we wouldn’t be able to narrow its return type.
```cp
func iden(v: unknown): unknown {
	return v;
}
let x: int = iden.(42); %> TypeError: `unknown` not assignable to `int`
```
Generic parameters also allow us to better control parameter types.
```cp
func eq1<T>(a: T, b: T): bool {
	return a == b;
}
eq1.<int>(42, '42'); %> TypeError
```
If we had used `unknown`, we wouldn’t have gotten a compile-time error.
```cp
func eq2(a: unknown, b: unknown): bool {
	return a == b;
}
eq2.(42, '42'); % no error; returns `false` at runtime
```

## Optional & Constrained Parameters
Like generic type aliases, generic functions can have optional generic parameters (with a default value). Below, if a type is not provided for `U`, it is `T` by default.
```cp
func foldList<T, U ?= T>(list: T[], reducer: (U, T) => U, initial: U): U {
	; % ...
}
let total: int = foldList.<int>([2, 3, 4], (a: int, b: int): int => a + b, 0);
```

Generic parameters that are not optional are required. Currently type inference is not supported.
```cp
func makeBox<T>(v: T): [T] => [v];
makeBox.('42'); %> TypeError: Got 0 type arguments, but expected 1.
```
Even though the compiler knows the type of the argument, it expects `makeBox` to be called with an explicit generic argument.

Generic parameters can also be constrained, using the `narrows` or `widens` keywords.
```cp
func makeBox<T narrows str?>(v: T): [T] => [v];
makeBox.<str>('42');  %: [str] %== ['42']
makeBox.<int>(42);    %> TypeError: Type `int` is not a subtype of type `str | null`.
makeBox.<str>(42);    %> TypeError: Expression of type `42` is not assignable to type `str`.
makeBox.<null>('42'); %> TypeError: Expression of type `'42'` is not assignable to type `null`.
```

## Generic Function Types
A **generic function type** has generic parameters preceding the regular parameter list.
```cp
type Comparator1 = <T>(T, T) => int;
```
The type above is a generic function that requires one type argument and two arguments every time it is called.
```cp
let lessThan1: Comparator1 = <U>(a: U, b: U): int {
	; % ...
}
lessThan1.<int>(3, 4);            %== true
lessThan1.<int>('three', 'four'); %== false
```

This is different from a *generic type alias that happens to be a function type*:
```cp
type Comparator2<T> = (T, T) => int;
```
The latter only requires a type argument when specified, but not when called.
```cp
let lessThan2: Comparator2.<float> = (a: float, b: float): int {
	; % ...
}
lessThan2.(3.0, 4.0); %== true
lessThan2.(3, 4);     %> TypeError: argument `int` not assignable to parameter `float`
```

It is possible to combine a *generic function type* with a *generic type alias*.
```cp
type Nuller<T> = <U>(T, U) => null;

let g: Nuller.<int> = <U>(i: int, j: U): null {
	i; j; return null;
};
let h: Nuller.<bool> = <U>(i: bool, j: U): null {
	i; j; return null;
};

g.<float>(42, 4.2);
h.<str>(true, 'world');
```

(To make matters worse, both concepts above are unrelated to *type functions* (#73).)
