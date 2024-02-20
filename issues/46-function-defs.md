Define synchronous function declarations and function expressions.

# Discussion

## Function Definitions
This issue covers defining void synchronous functions only, which do return but do not return a value (#70). This issue does not cover non-void functions, asynchronous functions, or error throwing.

This is a function declaration:
```cp
function add(a: int, b: int): void {
	"""
		The first argument is {{ a }}.
		The second argument is {{ b }}.
	""";
	"This function does not return a value.";
}
```
The **name** of this function is `add`. It has two input **parameters** `a` and `b`, each of type `int`. When a function is called (see v0.7.0), the values sent into the function are **arguments**. This function’s **return type** is `void`, because it has no output value. Its **body** is the set of statements within the curly braces.

This is a function expression (lambda):
```cp
(a: int, b: int): void {
	"""
		The first argument is {{ a }}.
		The second argument is {{ b }}.
	""";
	"This function does not return a value.";
};
```
Lambdas are normal expressions that can be operated on and passed around like any other value. For instance, lambdas can be assigned to variables. Lambdas are always “truthy”.
```cp
let my_fn: (a: int, b: int) => void =
	(a: int, b: int): void { a + b; };
!!my_fn; %== true
```

Some parameters may be declared with `var`, which means they can be reassigned within the function body.
```cp
function add(var a: int, b: int): void {
	a = a + 1; % ok
	b = b - 1; %> AssignmentError
}
```

## Type Signatures
A function’s **type signature** is its type, written in the form of `(‹params›) => ‹return›`. It indicates what types of arguments are accepted and what type the function returns. This issue only covers functions that return `void`.
```cp
let my_fn: (a: int, b: int) => void =
	(a: int, b: int): void { a + b; };

% typeof my_fn: (a: int, b: int) => void
```


## Function Assignment
When assigning a function to a type signature with named parameters (in the case of type alias assignment or abstract method implementation), the assigned parameter order must match up with the assignee parameters.
```cp
type BinaryOperatorType = (first: int, second: float) => void;
let add: BinaryOperatorType = (second: float, first: int): void { first + second; }; %> TypeError
```
> TypeError: Type `(second: float, first: int) => void` is not assignable to type `(first: int, second: float) => void`.

The reason for this error is that one should expect to be able to call any `BinaryOperatorType` with the positional arguments of an `int` followed by a `float`. Calling it with e.g. `4.0` and `2`, in that order, should fail. From this perspective, function assignment is a bit like tuple assignment.

From another perspective, function assignment is like record assignment: the parameter names of the assigned must match the parameter names of the assignee.
```cp
type BinaryOperatorType = (first: float, second: float) => void;
let subtract: BinaryOperatorType = (x: float, y: float): void { x - y; }; %> TypeError
```
> TypeError: Type `(x: float, y: float) => void` is not assignable to type `(first: float, second: float) => void`.

This errors because a caller must be able to call `subtract` with the named arguments `first` and `second`.

Luckily, function parameter syntax has a built-in mechanism for handling function assignment/implementation with named parameters. In the parameter name, use `first= x` to alias the real parameter `x` to the assignee parameter `first`.
```cp
let subtract: BinaryOperatorType = (first= x: float, second= y: float): void { x - y; };
```
This lets the function author internally use the parameter names `x` and `y` while still allowing the caller to call the function with named arguments `first` and `second` repectively.


## Variance
Function parameter types are **contravariant**. This means that when assigning a function `g` to a function type `F`, the type of each parameter of `F` must be assignable to the corresponding parameter’s type of `g`.
```cp
type UnaryOperator = (float | str) => void;
let g: UnaryOperator = (x: float): void { %> TypeError
	x; %: float
};
```
A type error is raised because we cannot assign a `(float) => void` type to a `(float | str) => void` type. Even though the parameter’s type is narrower, a caller should expect to be able to call any `UnaryOperator` implementation with a `str` argument, and our implementation doesn’t allow that.

However, we can *widen* the parameter types:
```cp
let h: UnaryOperator = (x: int | float | str): void {
	x; %: int | float | str
};
```
This meets the requirements of `UnaryOperator` but still has a wider type for its parameter.

# Specification

## Lexicon
```diff
Punctuator :::=
	// storage
+		| "=>"
;

Keyword :::=
	// storage
+		| "function"
;
```

## Syntax
```diff
+TypeFunction<Named>
+	::= "(" ","? ParameterType<?Named># ","? ")" "=>" "void";

Type ::=
	| TypeUnion
+	| TypeFunction<-Named, +Named>
;

+ExpressionFunction
+	::= "(" ","? ParameterFunction# ","? ")" ":" "void" StatementBlock<-Break>;

Expression<Dynamic> ::=
	| ExpressionDisjunctive<?Dynamic>
	| ExpressionConditional<?Dynamic>
+	| <Dynamic+>ExpressionFunction
;

+ParameterType<Named>
+	::= <Named+>(IDENTIFIER ":") Type;

+ParameterFunction
+	::= (IDENTIFIER "=")? "var"? IDENTIFIER ":" Type;

+DeclarationFunction
+	::= "function" IDENTIFIER ExpressionFunction;

Declaration ::=
	| DeclarationVariable
	| DeclarationType
+	| DeclarationFunction
;
```

## Semantics
```diff
SemanticType =:=
	| SemanticTypeConstant
	| SemanticTypeAlias
	| SemanticTypeEmptyCollection
	| SemanticTypeList
	| SemanticTypeRecord
	| SemanticTypeOperation
+	| SemanticTypeFunction
;

+SemanticTypeFunction
+	::= SemanticParameterType*;

SemanticExpression =:=
	| SemanticConstant
	| SemanticVariable
	| SemanticTemplate
	| SemanticEmptyCollection
	| SemanticList
	| SemanticRecord
	| SemanticMapping
	| SemanticOperation
+	| SemanticFunction
;

+SemanticFunction
+	::= SemanticParameter* SemanticBlock;

+SemanticParameterType
+	::= SemanticVariable? SemanticType;

+SemanticParameter[unfixed: Boolean]
+	::= SemanticKey? SemanticVariable SemanticType;

SemanticDeclaration =:=
	| SemanticDeclarationType
	| SemanticDeclarationVariable
+	| SemanticDeclarationFunction
;

+SemanticDeclarationFunction
+	::= SemanticVariable SemanticTypeFunction SemanticFunction;

SemanticBlock
	::= SemanticStatement*;
```

## Decorate
```diff
+Decorate(TypeFunction ::= "(" ","? ParameterType# ","? ")" "=>" "void") -> SemanticTypeFunction
+	:= (SemanticTypeFunction ...ParseList(ParameterType, SemanticParameterType));

+Decorate(Type ::= TypeFunction) -> SemanticTypeFunction
+	:= Decorate(TypeFunction);

+Decorate(ExpressionFunction ::= "(" ","? ParameterFunction# ","? ")" ":" "void" StatementBlock<-Break>) -> SemanticFunction
+	:= (SemanticFunction
+		...ParseList(ParameterFunction, SemanticParameter)
+		Decorate(StatementBlock<-Break>)
+	);

+Decorate(Expression_Dynamic ::= ExpressionFunction) -> SemanticFunction
+	:= Decorate(ExpressionFunction);

+Decorate(ParameterType ::= Type) -> SemanticParameterType
+	:= (SemanticParameterType Decorate(Type));
+Decorate(ParameterType_Named ::= IDENTIFIER ":" Type) -> SemanticParameterType
+	:= (SemanticParameterType
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);

+Decorate(ParameterFunction ::= IDENTIFIER ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);
+Decorate(ParameterFunction ::= "var" IDENTIFIER ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);
+Decorate(ParameterFunction ::= IDENTIFIER__0 "=" IDENTIFIER__1 ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticKey[id=TokenWorth(IDENTIFIER__0)])
+		(SemanticVariable[id=TokenWorth(IDENTIFIER__1)])
+		Decorate(Type)
+	);
+Decorate(ParameterFunction ::= IDENTIFIER__0 "=" "var" IDENTIFIER__1 ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticKey[id=TokenWorth(IDENTIFIER__0)])
+		(SemanticVariable[id=TokenWorth(IDENTIFIER__1)])
+		Decorate(Type)
+	);

+Decorate(DeclarationFunction ::= "func" IDENTIFIER ExpressionFunction) -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		FunctionTypeOf(ExpressionFunction)
+		Decorate(ExpressionFunction)
+	);

+Decorate(Declaration ::= DeclarationFunction) -> SemanticDeclarationFunction
+	:= Decorate(DeclarationFunction);
```

## FunctionTypeOf
```diff
+FunctionTypeOf(ExpressionFunction ::= "(" ","? ParameterFunction# ","? ")" ":" "void" StatementBlock<-Break>) -> SemanticTypeFunction
+	:= (SemanticTypeFunction ...FunctionTypeOf(ParameterFunction#));

+	FunctionTypeOf(ParameterFunction# ::= ParameterFunction) -> Tuple<SemanticParameterType>
+		:= [FunctionTypeOf(ParameterFunction)];
+	FunctionTypeOf(ParameterFunction# ::= ParameterFunction# "," ParameterFunction) -> Sequence<SemanticParameterType>
+		:= [
+			...FunctionTypeOf(ParameterFunction#),
+			FunctionTypeOf(ParameterFunction),
+		];

+FunctionTypeOf(ParameterFunction ::= "var"? IDENTIFIER ":" Type) -> SemanticParameterType
+	:= (SemanticParameterType
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);
+FunctionTypeOf(ParameterFunction ::= IDENTIFIER__0 "=" "var"? IDENTIFIER__1 ":" Type) -> SemanticParameterType
+	:= (SemanticParameterType
+		(SemanticVariable[id=TokenWorth(IDENTIFIER__0)])
+		Decorate(Type)
+	);
```
