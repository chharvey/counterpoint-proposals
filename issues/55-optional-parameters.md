**Optional parameters** allow arguments to be omitted from a function call.

# Discussion
An optional parameter must have a **default value**, and when the argument for that optional parameter is omitted, the function is called as if the default value were provided.

```point
function moveForward(steps: int ?= 1): void {
	steps; %: int
}
moveForward; %: \(steps?: int) => void
```
The code `?= 1` is called an **initializer** and defines the default value of the `steps` parameter. It’s similar to the initializer of a variable declaration. When `moveForward` is called without that argument, the default value of `1` is assumed.

Notice the type signature’s new syntax: `(steps?: int) => void`. This means that the function may be called with 0 or 1 argument, and if it is called with 1 argument, that argument *may* be given a name of `steps`, and it *must* be of type `int`. Function calls are not covered in this version.

The syntax of optional type parameters carries over into unnamed parameters of a type signature.
```point
type BinaryOperatorTypeUnnamed = \(float, ?:float) => void;
```
The signature above indicates that a function of type `BinaryOperatorTypeUnnamed` has 1 required parameter and 1 optional parameter. Optional parameters’ default values are an implemenation detail, so they cannot be specified in type signatures. And since the parameters of this type signature are unnamed, an implementing function can only be called with positional arguments.

All optional parameters must be declared after all required parameters; otherwise it’s a syntax error.
```point
function breakfast(entree: str, dessert: str ?= ""): void {;} % fine, type `\(entree: str, dessert?: str) => void`
function dinner   (dessert: str ?= "", entree: str): void {;} %> ParseError
type Meal = \(?:str, str) => void;                            %> ParseError
```
Specifying an initializer that mismatches the parameter type results in a TypeError.
```point
function moveForward(steps: int ?= false): void {;} %> TypeError: `false` not assignable to `int`
```

Optional parameters may be unfixed (reassignable within the function):
```point
function greet(var greeting: str ?= "Hello"): void {
	greeting = if greeting == "" then "Hi" else greeting;
	"""{{ greeting }}, world!""";
}
greet; %: \(greeting?: str) => void
```

## Default Parameter Evaluation
Optional parameter initializers are evaluated when the function is *called*, not when it’s *defined*.
```point
function say_hello(): void {
	print.("hello");
};
function run[say_hello](x: void ?= say_hello.()): void {}; % does not print "hello" here
run.();                                                    % prints "hello" here
run.();                                                    % prints "hello" again
```

If an optional parameter initializer references a variable, it must be captured (as shown above), and it refers to the variable bound to the environment in which it’s *initialized*, not in which the function is *called*. And, if that variable is ever reassigned outside the function, the reassignment is not observed. However, mutations will still be observed.
```point
%-- Variable Reassignment --%
%% line 2 %% let var init: bool = false;
function say[init](b: bool ?= init): void { print.(b); }
%                             ^ refers to the `init` from line 2
say.(); % prints `false`

set init = true; % reassigns `init` from line 2, but not captured variable on line 3
say.();          % still prints `false`
%   ^ reads from the value `false` in line 2
```
```point
%-- Import Shadowing --%
% Module "a"
%% line 3 %% let init: bool = false;
public function say[init](b: bool ?= init): void { print.(b); }
%                                    ^ refers to the `init` from line 3
say.(); % prints `false`

% Module "b"
from "a" import say;
let init: bool = true;
say.();                % still prints `false`
%   ^ reads from same `init` as Module "a" (not new `init` from Module "b")
```
```point
%-- Variable Mutation --%
let a: mut [int] = [42];
function twice[a](x: int ?= a.[0]): int => x * 2;
twice.(); %== 84
set a.[0] = 12;
twice.(); %== 24
```

# Specification

## Lexicon
```diff
Punctuator :::=
	// statement
-		| ";" | ":" | "?:" | "="
+		| ";" | ":" | "?:" | "=" | "?="
;
```

## Syntax
```diff
-TypeFunction<Named>
+TypeFunction
-	::= "\" "(" ","? ParameterType<?Named># ","? ")" "=>" ("void" | Type);
+	::= "\" "(" ","? ParametersType<∓Named>      ")" "=>" ("void" | Type);

Type ::=
	| TypeUnion
-	| TypeFunction<∓Named>
+	| TypeFunction
;

-DeclaredFunction   ::=     "(" ","? ParameterFunction# ","? ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression<+Block><-Break><+Return> ";");
-ExpressionFunction ::= "\" "(" ","? ParameterFunction# ","? ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression<+Block><-Break><+Return>);
+DeclaredFunction   ::=     "(" ","? ParametersFunction      ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression<+Block><-Break><+Return> ";");
+ExpressionFunction ::= "\" "(" ","? ParametersFunction      ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression<+Block><-Break><+Return>);

-ParameterType<Named>
+ParameterType<Named, Optional>
-	::= <Named+>(Word              ":")                 Type;
+	::= <Named+>(Word & <Optional->":") <Optional+>"?:" Type;

-ParameterFunction
+ParameterFunction<Optional>
-	::= (Word "=")? "var"? ("_" | IDENTIFIER) ":" Type;
+	::= (Word "=")? "var"? ("_" | IDENTIFIER) ":" Type & <Optional+>("?=" Expression<+Block><-Break><-Return>);

+ParametersType<Named> ::=
+	|  ParameterType<?Named><-Optional># ","?
+	| (ParameterType<?Named><-Optional># ",")? ParameterType<?Named><+Optional># ","?
+;

+ParametersFunction ::=
+	|  ParameterFunction<-Optional># ","?
+	| (ParameterFunction<-Optional># ",")? ParameterFunction<+Optional># ","?
+;
```

## Semantics
```diff
-SemanticParameterType
+SemanticParameterType[optional: Boolean]
	::= SemanticKey? SemanticType;

SemanticParameter[unfixed: Boolean]
-	::= SemanticKey? SemanticVariable SemanticType;
+	::= SemanticKey? SemanticVariable SemanticType SemanticExpression?;
```

## Decorate
```diff
-Decorate(TypeFunction<Named> ::= "\" "(" ","? ParameterType<?Named># ","? ")" "=>" "void") -> SemanticTypeFunction
+Decorate(TypeFunction        ::= "\" "(" ","? ParametersType<∓Named>      ")" "=>" "void") -> SemanticTypeFunction
-	:= (SemanticTypeFunction ...ParseList(ParameterType<?Named>, SemanticParameterType));
+	:= (SemanticTypeFunction ...Decorate(ParametersType<∓Named>));
-Decorate(TypeFunction<Named> ::= "\" "(" ","? ParameterType<?Named># ","? ")" "=>" Type) -> SemanticTypeFunction
+Decorate(TypeFunction        ::= "\" "(" ","? ParametersType<∓Named>#     ")" "=>" Type) -> SemanticTypeFunction
	:= (SemanticTypeFunction
-		...ParseList(ParameterType<?Named>, SemanticParameterType)
+		...Decorate(ParametersType<∓Named>)
		Decorate(Type)
	);

-Decorate(Type ::= TypeFunction<∓Named>) -> SemanticTypeFunction
+Decorate(Type ::= TypeFunction)         -> SemanticTypeFunction
-	:= Decorate(TypeFunction<∓Named>);
+	:= Decorate(TypeFunction);

-Decorate(DeclaredFunction ::= "(" ","? ParameterFunction# ","? ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
+Decorate(DeclaredFunction ::= "(" ","? ParametersFunction      ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
	:= (SemanticFunction
-		...ParseList(ParameterFunction, SemanticParameter)
+		...Decorate(ParametersFunction)
		Decorate(Block<-Break><+Return>)
	);
-Decorate(DeclaredFunction ::= "(" ","? ParameterFunction# ","? ")" ":" "void" "=>" Expression<+Block><-Break><+Return> ";") -> SemanticFunction
+Decorate(DeclaredFunction ::= "(" ","? ParametersFunction      ")" ":" "void" "=>" Expression<+Block><-Break><+Return> ";") -> SemanticFunction
	:= (SemanticFunction
-		...ParseList(ParameterFunction, SemanticParameter)
+		...Decorate(ParametersFunction)
		Decorate(Expression<+Block><-Break><+Return>)
	);
-Decorate(DeclaredFunction ::= "(" ","? ParameterFunction# ","? ")" ":" Type Block<-Break><+Return>) -> SemanticFunction
+Decorate(DeclaredFunction ::= "(" ","? ParametersFunction      ")" ":" Type Block<-Break><+Return>) -> SemanticFunction
	:= (SemanticFunction
-		...ParseList(ParameterFunction, SemanticParameter)
+		...Decorate(ParametersFunction)
		Decorate(Type)
		Decorate(Block<-Break><+Return>)
	);
-Decorate(DeclaredFunction ::= "(" ","? ParameterFunction# ","? ")" ":" Type "=>" Expression<+Block><-Break><+Return> ";") -> SemanticFunction
+Decorate(DeclaredFunction ::= "(" ","? ParametersFunction      ")" ":" Type "=>" Expression<+Block><-Break><+Return> ";") -> SemanticFunction
	:= (SemanticFunction
-		...ParseList(ParameterFunction, SemanticParameter)
+		...Decorate(ParametersFunction)
		Decorate(Type)
		Decorate(Expression<+Block><-Break><+Return>)
	);



-Decorate(ExpressionFunction ::= "\" "(" ","? ParameterFunction# ","? ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
+Decorate(ExpressionFunction ::= "\" "(" ","? ParametersFunction      ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
	:= (SemanticFunction
-		...ParseList(ParameterFunction, SemanticParameter)
+		...Decorate(ParametersFunction)
		Decorate(Block<-Break><+Return>)
	);
-Decorate(ExpressionFunction ::= "\" "(" ","? ParameterFunction# ","? ")" ":" "void" "=>" Expression<+Block><-Break><+Return>) -> SemanticFunction
+Decorate(ExpressionFunction ::= "\" "(" ","? ParametersFunction      ")" ":" "void" "=>" Expression<+Block><-Break><+Return>) -> SemanticFunction
	:= (SemanticFunction
-		...ParseList(ParameterFunction, SemanticParameter)
+		...Decorate(ParametersFunction)
		Decorate(Expression<+Block><-Break><+Return>)
	);
-Decorate(ExpressionFunction ::= "\" "(" ","? ParameterFunction# ","? ")" ":" Type Block<-Break><+Return>) -> SemanticFunction
+Decorate(ExpressionFunction ::= "\" "(" ","? ParametersFunction      ")" ":" Type Block<-Break><+Return>) -> SemanticFunction
	:= (SemanticFunction
-		...ParseList(ParameterFunction, SemanticParameter)
+		...Decorate(ParametersFunction)
		Decorate(Type)
		Decorate(Block<-Break><+Return>)
	);
-Decorate(ExpressionFunction ::= "\" "(" ","? ParameterFunction# ","? ")" ":" Type "=>" Expression<+Block><-Break><+Return>) -> SemanticFunction
+Decorate(ExpressionFunction ::= "\" "(" ","? ParametersFunction      ")" ":" Type "=>" Expression<+Block><-Break><+Return>) -> SemanticFunction
	:= (SemanticFunction
-		...ParseList(ParameterFunction, SemanticParameter)
+		...Decorate(ParametersFunction)
		Decorate(Type)
		Decorate(Expression<+Block><-Break><+Return>)
	);

Decorate(ParameterType ::= Type) -> SemanticParameterType
-	:= (SemanticParameterType Decorate(Type));
+	:= (SemanticParameterType[optional=false] Decorate(Type));
+Decorate(ParameterType_Optional ::= "?:" Type) -> SemanticParameterType
+	:= (SemanticParameterType[optional=true] Decorate(Type));
Decorate(ParameterType_Named ::= Word ":" Type) -> SemanticParameterType
-	:= (SemanticParameterType
+	:= (SemanticParameterType[optional=false]
		Decorate(Word)
		Decorate(Type)
	);
+Decorate(ParameterType_Named_Optional ::= Word "?:" Type) -> SemanticParameterType
+	:= (SemanticParameterType[optional=true]
+		Decorate(Word)
+		Decorate(Type)
+	);

-Decorate(ParameterFunction            ::= "_" ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Optional> ::= "_" ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		Decorate(Type)
	);
-Decorate(ParameterFunction            ::= IDENTIFIER ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Optional> ::= IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
-Decorate(ParameterFunction            ::= "var" "_" ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Optional> ::= "var" "_" ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		Decorate(Type)
	);
-Decorate(ParameterFunction            ::= "var" IDENTIFIER ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Optional> ::= "var" IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
-Decorate(ParameterFunction            ::= Word "=" "_" ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Optional> ::= Word "=" "_" ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		Decorate(Word)
		Decorate(Type)
	);
-Decorate(ParameterFunction            ::= Word "=" IDENTIFIER ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Optional> ::= Word "=" IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		Decorate(Word)
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
-Decorate(ParameterFunction            ::= Word "=" "var" "_" ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Optional> ::= Word "=" "var" "_" ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		Decorate(Word)
		Decorate(Type)
	);
-Decorate(ParameterFunction            ::= Word "=" "var" IDENTIFIER ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Optional> ::= Word "=" "var" IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		Decorate(Word)
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
+Decorate(ParameterFunction<+Optional> ::= "_" ":" Type "?=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Optional> ::= IDENTIFIER ":" Type "?=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Optional> ::= "var" "_" ":" Type "?=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Optional> ::= "var" IDENTIFIER ":" Type "?=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Optional> ::= Word "=" "_" ":" Type "?=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		Decorate(Word)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Optional> ::= Word "=" IDENTIFIER ":" Type "?=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		Decorate(Word)
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Optional> ::= Word "=" "var" "_" ":" Type "?=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		Decorate(Word)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Optional> ::= Word "=" "var" IDENTIFIER ":" Type "?=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		Decorate(Word)
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);

+Decorate(ParametersType<Named> ::= ParameterType<?Named><-Optional># ","?) -> Sequence<SemanticParameterType>
+	:= ParseList(ParameterType<?Named><-Optional>, SemanticParameterType)
+Decorate(ParametersType<Named> ::= ParameterType<?Named><+Optional># ","?) -> Sequence<SemanticParameterType>
+	:= ParseList(ParameterType<?Named><+Optional>, SemanticParameterType)
+Decorate(ParametersType<Named> ::= ParameterType<?Named><-Optional># "," ParameterType<?Named><+Optional># ","?) -> Sequence<SemanticParameterType>
+	:= [
+		...ParseList(ParameterType<?Named><-Optional>, SemanticParameterType),
+		...ParseList(ParameterType<?Named><+Optional>, SemanticParameterType),
+	];

+Decorate(ParametersFunction ::= ParameterFunction<-Optional># ","?) -> Sequence<SemanticParameter>
+	:= ParseList(ParameterFunction<-Optional>, SemanticParameter);
+Decorate(ParametersFunction ::= ParameterFunction<+Optional># ","?) -> Sequence<SemanticParameter>
+	:= ParseList(ParameterFunction<+Optional>, SemanticParameter);
+Decorate(ParametersFunction ::= ParameterFunction<-Optional># "," ParameterFunction<+Optional># ","?) -> Sequence<SemanticParameter>
+	:= [
+		...ParseList(ParameterFunction<-Optional>, SemanticParameter),
+		...ParseList(ParameterFunction<+Optional>, SemanticParameter),
+	];
```

## FunctionTypeOf
```diff
-FunctionTypeOf(ExpressionFunction ::= "\" "(" ","? ParameterFunction# ","? ")" ":" "void" Block<-Break>) -> SemanticTypeFunction
-	:= (SemanticTypeFunction ...FunctionTypeOf(ParameterFunction#));
+FunctionTypeOf(ExpressionFunction ::= "\" "(" ","? ParametersFunction      ")" ":" "void" Block<-Break>) -> SemanticTypeFunction
+	:= (SemanticTypeFunction ...FunctionTypeOf(ParametersFunction));

+	FunctionTypeOf(ParametersFunction ::= ParameterFunction<-Optional># ","?) -> Sequence<SemanticParameter>
+		:= FunctionTypeOf(ParameterFunction<-Optional>#);
+	FunctionTypeOf(ParametersFunction ::= ParameterFunction<+Optional># ","?) -> Sequence<SemanticParameter>
+		:= FunctionTypeOf(ParameterFunction<+Optional>#);
+	FunctionTypeOf(ParametersFunction ::= ParameterFunction<-Optional># "," ParameterFunction<+Optional># ","?) -> Sequence<SemanticParameter>
+		:= [
+			...FunctionTypeOf(ParameterFunction<-Optional>#),
+			...FunctionTypeOf(ParameterFunction<+Optional>#),
+		];

-	FunctionTypeOf(ParameterFunction#            ::= ParameterFunction)            -> Tuple<SemanticParameterType>
+	FunctionTypeOf(ParameterFunction<±Optional># ::= ParameterFunction<±Optional>) -> Tuple<SemanticParameterType>
-		:= [FunctionTypeOf(ParameterFunction)];
+		:= [FunctionTypeOf(ParameterFunction<±Optional>)];
-	FunctionTypeOf(ParameterFunction#            ::= ParameterFunction#            "," ParameterFunction)            -> Sequence<SemanticParameterType>
+	FunctionTypeOf(ParameterFunction<±Optional># ::= ParameterFunction<±Optional># "," ParameterFunction<±Optional>) -> Sequence<SemanticParameterType>
		:= [
-			...FunctionTypeOf(ParameterFunction#),
-			FunctionTypeOf(ParameterFunction),
+			...FunctionTypeOf(ParameterFunction<±Optional>#),
+			FunctionTypeOf(ParameterFunction<±Optional>),
		];

-FunctionTypeOf(ParameterFunction            ::= "var"? ("_" | IDENTIFIER) ":" Type)                                -> SemanticParameterType
+FunctionTypeOf(ParameterFunction<±Optional> ::= "var"? ("_" | IDENTIFIER) ":" Type & <Optional+>("?=" Expression)) -> SemanticParameterType
	:= (SemanticParameterType
		(SemanticKey[id=TokenWorth("_" | IDENTIFIER)])
		Decorate(Type)
	);
-FunctionTypeOf(ParameterFunction            ::= Word "=" "var"? ("_" | IDENTIFIER) ":" Type)                                -> SemanticParameterType
+FunctionTypeOf(ParameterFunction<±Optional> ::= Word "=" "var"? ("_" | IDENTIFIER) ":" Type & <Optional+>("?=" Expression)) -> SemanticParameterType
	:= (SemanticParameterType
		Decorate(Word)
		Decorate(Type)
	);
```
