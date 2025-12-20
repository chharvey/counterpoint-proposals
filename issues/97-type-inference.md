Type inference allows the programmer to omit explicit type annotations from some declarations and rely on the compiler’s type inference system. There are two kinds of inference: “bottom-up” typing, which involves inferring the type of a target symbol based on its assigned expression, and “top-down” typing, which involves inferring the type of an assigned expression based on its target symbol’s declared type.

This issue opens the discussion, but not all features are slated for v0.5.0. Specifically, only variable declarations are slated for this release. Other features such as classes and functions are just brainstormed here.

# Bottom-Up Type Inference
If a variable/field is initialized to a primitive literal, template literal, typed function expression, or constructor call (including “IICEs”: immediately-invoked class expressions), then the type annotation may be omitted and the symbol’s type is inferred by the compiler. This also applies to optional function parameters with explicit default values.

For fixed variables and constant fields, the inferred type is the narrowest type of the initializer. A primitive literal implies a unit type, a template literal implies the `str` type, a typed function expression implies its type, and a constructor call implies the corresponding interface of the constructor’s class (mutable if not a data class).
```cpl
val untyped_var = null;                 %: null
val untyped_var = false;                %: false
val untyped_var = @hello;               %: @hello
val untyped_var = 42;                   %: 42
val untyped_var = 4.2;                  %: 4.2
val untyped_var = "hello";              %: "hello"
val untyped_var = """{{ 4 }}{{ 2 }}"""; %: str

class Point {
	public const x = 3.0; %: 3.0
	public const y = 4.0; %: 4.0
}

val untyped_var = \(a: int, b: int): int => a + b; %: (a: int, b: int) => int
val untyped_var = Vector.(1.1, 2.2);               %: Vector     % assuming it’s a data class
val untyped_var = Person.("Doe", "John");          %: mut Person % assuming it’s not a data class

val untyped_var = (class data {
	new (
		public x: float,
		public y: float,
	) {}
}).(1.1, 2.2); %: interface { readonly x: float; readonly y: float; } % anonymous class with public members

val untyped_var = (class data {
	new (private const name: str) {}
}).("world"); %: anything % anonymous data class with no public members

val untyped_var = (class {
	new (private name: str) {}
}).("world"); %: mut Object % anonymous reference class with no public members
```
To declare a variable as a non-mutable type of its initializer’s class constructor, we have to duplicate the type name in the type annotation. There may be plans to address this in the future.
```cpl
val untyped_var: Person = Person.("Doe", "John");
%                ^ redundant, but necessary if we want a non-mutable type
set untyped_var.lastName = "Smith"; %> MutabilityError
```

For unfixed variables and non-constant fields (even if `readonly`), if the initializer/default value is a primitive literal, then the inferred type is widened to one of the primitive types `bool`, `sym`, `int`, `float`, or `str` corresponding to the primitive literal. Otherwise the type is inferred in the same way as above.
```cpl
val mut untyped_unfixed_var = null;                 %: null
val mut untyped_unfixed_var = false;                %: bool
val mut untyped_unfixed_var = @hello;               %: sym
val mut untyped_unfixed_var = 42;                   %: int
val mut untyped_unfixed_var = 4.2;                  %: float
val mut untyped_unfixed_var = "hello";              %: str
val mut untyped_unfixed_var = """{{ 4 }}{{ 2 }}"""; %: str

class Point {
	public readonly x = 0.0; %: float
	public readonly y = 0.0; %: float
}
```

If the symbol is uninitialized or initialized to something else (e.g. a variable, operation, property access, function call, block-expression, or *untyped* lambda), then the type annotation is required. An exception is made for function parameters in certain situations, which is discussed in the next section.
```cpl
val untyped_var;             %> ParseError
val mut untyped_unfixed_var; %> ParseError

val untyped_var = null && true;     %> AssignmentError
val untyped_var = !!false;          %> AssignmentError
val untyped_var = @hi || @there;    %> AssignmentError
val untyped_var = 41 + 1;           %> AssignmentError
val untyped_var = 3.2 + 1.5;        %> AssignmentError
val untyped_var = return_hello.();  %> AssignmentError
val untyped_var = ("hello",).0;     %> AssignmentError
val untyped_var = \(a, b) => a + b; %> AssignmentError

func voidFn(no_default_value) {;}                       %> AssignmentError
func sum(x? = 1 - 1, mut y? = parseInt.("0")) => x + y; %> AssignmentError
```

If the symbol is declared/initialized to a collection literal (tuple/record/list/dict/set/map), then the above rules are applied recursively based on that collection’s entries. For example, if the collection only contains primitive literals, typed lambdas, and constructor calls, then the symbol may be unannotated. If the collection literal contains variables, operations, property accesses, or function calls, then the symbol must be explicitly annotated.
```cpl
val untyped_var = (
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

val mut untyped_unfixed_var = (
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

val untyped_var = (41 + 1,);           %> AssignmentError
val untyped_var = (hello,);            %> AssignmentError
val untyped_var = (return_hello.(),);  %> AssignmentError
val untyped_var = (\(a, b) => a + b,); %> AssignmentError
```

# Top-Down Type Inference
Top-down type inference is used for inferring parameter and return types of a function expression when it’s assigned to a symbol (a variable, field, collection entry, or typed function parameter). When this is the case, we may omit type annotations from the function’s parameters and return signature, even if its parameters are required (or don’t have default values).
```cpl
val typed_var: \(a: int, b: int) => int = \(a, b) {
	a; %: int
	b; %: int
	return a + b;
};

type BinaryOperation<T> = \(a: T, b: T) => T;
val ops: List.<BinaryOperation.<float>> = [
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

As shown in #84, an untyped optional parameter is succeeded by a question mark `?` (and potentially a default value).
```cpl
type Binop = \(float, ?: float) => float; % second parameter is optional
val add: Binop = \(a, b?) {
%                     ^ optional
	a; %: float
	b; %: float | null
	return a + (b || 0.0);
};
add.(2.0, 3.0); %== 5.0
add.(2.0);      %== 2.0
```

In the last section, we said that a parameter initialized to something other than a literal, typed lambda, or constructor call, would require a type annotation. The exception is in top-down typing, where an untyped lambda has a parameter with a default value. The default value can be “something else” (e.g. a variable, operation, property access, function call, block-expression, or even another *untyped* lambda), which is acceptable because the whole lambda is getting typed anyway, through top-down assignment.
```cpl
val subtract: \(int, ?:int) => int = \(a, b? = some_value) => a - b;
%                                              ^ type inference is still possible here because `b` will be assigned type `int`.
type Binop = \(int, ?:int) => int;
func add(a, b? = some_value) impl Binop => a + b;
%                ^ same… via the `impl` clause
```

Another form of top-down type inference applies to generic arguments in constructor calls. When a constructor call expression is assigned to a symbol with an explicit type, we can omit the generic arguments from the call expression.
```cpl
class Box<T> {
	new (public value: T) {;}
}

val data = Box.<int>(42); % bottom-up type inference (explained in last section)

val data: Box.<int> = Box.(42); % top-down type inference
%                        ^ constructor call expression may omit the `<int>` generic arg

val data: Box.<int> | Box.<float> = Box.(42); %> TypeError: generic argument required

func make_box<U>(value: U): mut Box.<U>
	=> Box.<U>(value);

val data: Box.<int> = make_box.(42); %> TypeError: generic argument required
```
Note that the annotated type must be the same as the constructor’s return type, or a non-`mut` version of it. This top-down generic inference only applies to *constructor* call expressions, not arbitrary function call expressions.
