Type inference allows the programmer to omit explicit type annotations from some declarations and rely on the compiler’s type inference system. There are two kinds of inference: “bottom-up” typing, which involves inferring the type of a target symbol based on its assigned expression, and “top-down” typing, which involves inferring the type of an assigned expression based on its target symbol’s declared type.

This issue opens the discussion, but not all features are slated for v0.5.0. Specifically, only variable declarations are slated for this release. Other features such as classes and functions are just brainstormed here.

# Bottom-Up Type Inference
If a variable/field is initialized to a primitive literal, template literal, typed lambda, or constructor call (including “IICEs”: immediately-invoked class expressions), then the type annotation may be omitted and the symbol’s type is inferred by the compiler. This also applies to optional function parameters with explicit default values.

For fixed variables and constant fields, the inferred type is the narrowest type of the initializer. A primitive literal implies a unit type, a template literal implies the `str` type, a typed lambda implies its type, and a constructor call implies the corresponding interface of the constructor’s class (mutable if not a data class).
```cp
let untyped_var = null;                 %: null
let untyped_var = false;                %: false
let untyped_var = 42;                   %: 42
let untyped_var = 4.2;                  %: 4.2
let untyped_var = "hello";              %: "hello"
let untyped_var = """{{ 4 }}{{ 2 }}"""; %: str

class Point {
	public const x = 0.0; %: 0.0
	public const y = 0.0; %: 0.0
}

let untyped_var = (a: int, b: int): int => a + b; %: (a: int, b: int) => int
let untyped_var = Point.(1.1, 2.2);               %: Point      % assuming it’s a data class
let untyped_var = Person.("Doe", "John");         %: mut Person % assuming it’s not a data class

let untyped_var = (class data {
	new (
		public x: float,
		public y: float,
	) {}
}).(1.1, 2.2); %: interface { readonly x: float, readonly y: float }

let untyped_var = (class {
	new (private name: str) {}
}).("world"); %: mut Object
```

For unfixed variables, non-constant fields (even if `readonly`), and all function parameters, if the initializer/default value is a primitive literal, then the inferred type is widened to one of the primitive types `bool`, `int`, `float`, or `str` corresponding to the primitive literal. Otherwise the type is inferred in the same way as above.
```cp
let var untyped_unfixed_var = null;                 %: null
let var untyped_unfixed_var = false;                %: bool
let var untyped_unfixed_var = 42;                   %: int
let var untyped_unfixed_var = 4.2;                  %: float
let var untyped_unfixed_var = "hello";              %: str
let var untyped_unfixed_var = """{{ 4 }}{{ 2 }}"""; %: str

class Point {
	public readonly x = 0.0; %: float
	public readonly y = 0.0; %: float
}

function sum(x ?= 0, var y ?= 0): int {
	x; %: int
	y; %: int
	return x + y;
}
```

If the symbol is uninitialized or initialized to something else (e.g. a variable, operation, property access, function call, or *untyped* lambda), then the type annotation is required. An exception is made for function parameters in certain situations, which are discussed in the next section.
```cp
let untyped_var;             %> ParseError
let var untyped_unfixed_var; %> ParseError

let untyped_var = null && true;    %> TypeError
let untyped_var = !!false;         %> TypeError
let untyped_var = 41 + 1;          %> TypeError
let untyped_var = 3.2 + 1;         %> TypeError
let untyped_var = return_hello.(); %> TypeError
let untyped_var = ["hello"].0;     %> TypeError
let untyped_var = (a, b) => a + b; %> TypeError

function voidFn(no_default_value): void {}                       %> TypeError
function sum(x ?= 1 - 1, var y ?= parseInt.("0")): int => x + y; %> TypeError
```

If the symbol is declared/initialized to a collection literal (tuple/record), then the above rules are applied recursively based on that collection’s entries. For example, if the collection only contains primitive literals, typed lambdas, and constructor calls, then the symbol may be unannotated. If the collection literal contains variables, operations, property accesses, or function calls, then the symbol must be explicitly annotated.
```cp
let untyped_var = [
	null,
	false,
	42,
	4.2,
	"hello",
	"""{{ 4 }}{{ 2 }}""",
	(a: int, b: int): int => a + b,
]; %%: [
	null,
	false,
	42,
	4.2,
	"hello",
	str,
	(a: int, b: int) => int,
] %%

let var untyped_unfixed_var = [
	null,
	false,
	42,
	4.2,
	"hello",
	"""{{ 4 }}{{ 2 }}""",
	(a: int, b: int): int => a + b,
]; %%: [
	null,
	bool,
	int,
	float,
	str,
	str,
	(a: int, b: int) => int,
] %%

let untyped_var = [41 + 1];          %> TypeError
let untyped_var = [hello];           %> TypeError
let untyped_var = [return_hello.()]; %> TypeError
let untyped_var = [(a, b) => a + b]; %> TypeError
```

# Top-Down Type Inference
Top-down type inference is used for inferring parameter and return types of a lambda when the lambda is assigned to a symbol (a variable, field, collection entry, or function parameter). When this is the case, we may omit type annotations from the lambda’s parameters and return signature, even if its parameters are required (don’t have default values).
```cp
let typed_var: (a: int, b: int) => int = (a, b) {
	a; %: int
	b; %: int
	return a + b;
};

type BinaryOperation<T> = (a: T, b: T) => T;
let ops: List.<BinaryOperation.<float>> = List.<BinaryOperation.<float>>([
	(a, b) {
		a;            %: float
		b;            %: float
		return a + b; %: float
	},
]);
% (Note we could have also omitted the variable type annotation above, utilizing “bottom-up” inference.)

claim map<T, U>(list: T[], mapper: (T, int) => U): U[];
claim people: Person[];

map.<Person, str>(people, (p, i) {
	p;                 %: Person
	i;                 %: int
	return p.fullName; %: str
});
```

Note that when type annotations are omitted from optional parameters, they syntactically look like required parameters. They’re still optional though.
```cp
type Binop = (int, ?: int) => int;
function add(a, b) impl Binop {
%               ^ looks required, but isn’t
	a; %: int
	b; %: int?
	return a + (b || 0);
};
add.(2, 3); %== 5
add.(2);    %== 2
```
In this example, `add` implements `Binop`, so we don’t explicitly write out its parameter types. Parameter `b` is optional, but it also doesn’t have an explicit default value (its default value is implicitly `null` — see #55). These facts combined make it look like `b` is a required parameter from a syntax perspective. Semantically, though, it’s still optional when the function is called.

It is better to provide a default value if possible.
```cp
type Binop = (int, ?: int) => int;
function add(a, b ?= 0) impl Binop {
	a; %: int
	b; %: int
	return a + b;
};
add.(2, 3); %== 5
add.(2);    %== 2
```
