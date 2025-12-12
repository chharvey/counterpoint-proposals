Type inference allows the programmer to omit explicit type annotations from some declarations and rely on the compiler’s type inference system. There are two kinds of inference: “bottom-up” typing, which involves inferring the type of a target symbol based on its assigned expression, and “top-down” typing, which involves inferring the type of an assigned expression based on its target symbol’s declared type.

This issue opens the discussion, but not all features are slated for v0.5.0. Specifically, only variable declarations are slated for this release. Other features such as classes and functions are just brainstormed here.

# Bottom-Up Type Inference
If a variable/field is initialized to a primitive literal, template literal, typed lambda, or constructor call (including “IICEs”: immediately-invoked class expressions), then the type annotation may be omitted and the symbol’s type is inferred by the compiler. This also applies to optional function parameters with explicit default values.

For fixed variables and constant fields, the inferred type is the narrowest type of the initializer. A primitive literal implies a unit type, a template literal implies the `str` type, a typed lambda implies its type, and a constructor call implies the corresponding interface of the constructor’s class (mutable if not a data class).
```cpl
let untyped_var = null;                 %: null
let untyped_var = false;                %: false
let untyped_var = @hello;               %: @hello
let untyped_var = 42;                   %: 42
let untyped_var = 4.2;                  %: 4.2
let untyped_var = "hello";              %: "hello"
let untyped_var = """{{ 4 }}{{ 2 }}"""; %: str

class Point {
	public const x = 3.0; %: 3.0
	public const y = 4.0; %: 4.0
}

let untyped_var = \(a: int, b: int): int => a + b; %: (a: int, b: int) => int
let untyped_var = Vector.(1.1, 2.2);               %: Vector     % assuming it’s a data class
let untyped_var = Person.("Doe", "John");          %: mut Person % assuming it’s not a data class

let untyped_var = (class data {
	new (
		public x: float,
		public y: float,
	) {}
}).(1.1, 2.2); %: interface { readonly x: float; readonly y: float; } % anonymous class with public members

let untyped_var = (class data {
	new (private const name: str) {}
}).("world"); %: anything % anonymous data class with no public members

let untyped_var = (class {
	new (private name: str) {}
}).("world"); %: mut Object % anonymous reference class with no public members
```
To declare a variable as a non-mutable type of its initializer’s class constructor, we have to duplicate the type name in the type annotation. There may be plans to address this in the future.
```cpl
let untyped_var: Person = Person.("Doe", "John");
%                ^ redundant, but necessary if we want a non-mutable type
set untyped_var.lastName = "Smith"; %> MutabilityError
```

For unfixed variables and non-constant fields (even if `readonly`), if the initializer/default value is a primitive literal, then the inferred type is widened to one of the primitive types `bool`, `sym`, `int`, `float`, or `str` corresponding to the primitive literal. Otherwise the type is inferred in the same way as above.
```cpl
let var untyped_unfixed_var = null;                 %: null
let var untyped_unfixed_var = false;                %: bool
let var untyped_unfixed_var = @hello;               %: sym
let var untyped_unfixed_var = 42;                   %: int
let var untyped_unfixed_var = 4.2;                  %: float
let var untyped_unfixed_var = "hello";              %: str
let var untyped_unfixed_var = """{{ 4 }}{{ 2 }}"""; %: str

class Point {
	public readonly x = 0.0; %: float
	public readonly y = 0.0; %: float
}
```

If the symbol is uninitialized or initialized to something else (e.g. a variable, operation, property access, function call, block-expression, or *untyped* lambda), then the type annotation is required. An exception is made for function parameters in certain situations, which is discussed in the next section.
```cpl
let untyped_var;             %> ParseError
let var untyped_unfixed_var; %> ParseError

let untyped_var = null && true;     %> TypeError
let untyped_var = !!false;          %> TypeError
let untyped_var = @hi || @there;    %> TypeError
let untyped_var = 41 + 1;           %> TypeError
let untyped_var = 3.2 + 1.5;        %> TypeError
let untyped_var = return_hello.();  %> TypeError
let untyped_var = ("hello",).0;     %> TypeError
let untyped_var = \(a, b) => a + b; %> TypeError

function voidFn(no_default_value) {;}                       %> TypeError
function sum(x ?= 1 - 1, var y ?= parseInt.("0")) => x + y; %> TypeError
```

If the symbol is declared/initialized to a collection literal (tuple/record/list/dict/set/map), then the above rules are applied recursively based on that collection’s entries. For example, if the collection only contains primitive literals, typed lambdas, and constructor calls, then the symbol may be unannotated. If the collection literal contains variables, operations, property accesses, or function calls, then the symbol must be explicitly annotated.
```cpl
let untyped_var = (
	null,
	false,
	42,
	4.2,
	"hello",
	"""{{ 4 }}{{ 2 }}""",
	\(a: int, b: int): int => a + b,
); %%: (
	null,
	false,
	42,
	4.2,
	"hello",
	str,
	\(a: int, b: int) => int,
) %%

let var untyped_unfixed_var = (
	null,
	false,
	42,
	4.2,
	"hello",
	"""{{ 4 }}{{ 2 }}""",
	\(a: int, b: int): int => a + b,
); %%: (
	null,
	bool,
	int,
	float,
	str,
	str,
	\(a: int, b: int) => int,
) %%

let untyped_var = (41 + 1,);           %> TypeError
let untyped_var = (hello,);            %> TypeError
let untyped_var = (return_hello.(),);  %> TypeError
let untyped_var = (\(a, b) => a + b,); %> TypeError
```

# Top-Down Type Inference
Top-down type inference is used for inferring parameter and return types of a lambda when the lambda is assigned to a symbol (a variable, field, collection entry, or typed function parameter). When this is the case, we may omit type annotations from the lambda’s parameters and return signature, even if its parameters are required (or don’t have default values).
```cpl
let typed_var: \(a: int, b: int) => int = \(a, b) {
	a; %: int
	b; %: int
	return a + b;
};

type BinaryOperation<T> = \(a: T, b: T) => T;
let ops: List.<BinaryOperation.<float>> = [
	\(a, b) {
		a;            %: float
		b;            %: float
		return a + b; %: float
	},
];

claim map<T, U>(list: [T], mapper: \(T, int) => U): [U];
claim people: [Person];

map.<Person, str>(people, \(p, i) {
	p;                 %: Person
	i;                 %: int
	return p.fullName; %: str
});
```

As discussed in #84, there’s not currently a way for an untyped function expression to indicate that one of its parameters is optional, aside from providing a default value. There may be a future change in syntax to address this problem.
```cpl
type Binop = \(float, ?: float) => float; % second parameter is optional
let add: Binop = \(a, b) {
%                     ^ looks required, but isn’t … providing a default value would make it clear
	a; %: float
	b; %: float | null
	return a + (b || 0.0);
};
add.(2.0, 3.0); %== 5.0
add.(2.0);      %== 2.0
```

In the last section, we said that a parameter initialized to something other than a literal, typed lambda, or constructor call, would require a type annotation. The exception is in top-down typing, where an untyped lambda has a parameter with a default value. The default value can be “something else” (e.g. a variable, operation, property access, function call, block-expression, or even another *untyped* lambda), which is acceptable because the whole lambda is getting typed anyway, through top-down assignment.
```cpl
let subtract: \(int, ?: int) => int = (a, b ?= some_value) => a - b;
%                                              ^ type inference is still possible here because `b` will be assigned type `int`.
type Binop = \(int, ?: int) => int;
function add(a, b ?= some_value) impl Binop => a + b;
%                    ^ same… via the `impl` clause
```
