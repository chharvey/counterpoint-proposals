**Optional parameters** allow arguments to be omitted from a function call.

# Discussion
An optional parameter must have a **default value**, and when the argument for that optional parameter is omitted, the function is called as if the default value were provided.

```cp
function moveForward(steps: int ?= 1): void {
	steps; %: int
}
moveForward; %: (steps?: int) => void
```
The code `?= 1` is called an **initializer** and defines the default value of the `steps` parameter. It’s similar to the initializer of a variable declaration. When `moveForward` is called without that argument, the default value of `1` is assumed.

Notice the type signature’s new syntax: `(steps?: int) => void`. This means that the function may be called with 0 or 1 argument, and if it is called with 1 argument, that argument *may* be given a name of `steps`, and it *must* be of type `int`. Function calls are not covered in this version.

The syntax of optional type parameters carries over into unnamed parameters of a type signature.
```cp
type BinaryOperatorTypeUnnamed = (float, ?:float) => void;
```
The signature above indicates that a function of type `BinaryOperatorTypeUnnamed` has 1 required parameter and 1 optional parameter. Optional parameters’ default values are an implemenation detail, so they cannot be specified in type signatures. And since the parameters of this type signature are unnamed, an implementing function can only be called with positional arguments.

All optional parameters must be declared after all required parameters; otherwise it’s a syntax error.
```cp
function breakfast(entree: str, dessert: str ?= ""): void {;} % fine, type `(entree: str, dessert?: str) => void`
function dinner   (dessert: str ?= "", entree: str): void {;} %> ParseError
type Meal = (?:str, str) => void;                             %> ParseError
```
Specifying an initializer that mismatches the parameter type results in a TypeError.
```cp
function moveForward(steps: int ?= false): void {;} %> TypeError: `false` not assignable to `int`
```

Optional parameters may be unfixed (reassignable within the function):
```cp
function greet(var greeting: str ?= "Hello"): void {
	greeting = if greeting == "" then "Hi" else greeting;
	"""{{ greeting }}, world!""";
}
greet; %: (greeting?: str) => void
```

## Default Parameter Evaluation
Optional parameter initializers are evaluated when the function is *called*, not when it’s *defined*.
```cp
function say_hello(): void {
	print.("hello");
};
function run[say_hello](x: void ?= say_hello.()): void {}; % does not print "hello" here
run.();                                                    % prints "hello" here
run.();                                                    % prints "hello" again
```

If an optional parameter initializer references a variable, it must be captured (as shown above), and it refers to the variable bound to the environment in which it’s *initialized*, not in which the function is *called*. And, if that variable is ever reassigned outside the function, the reassignment is not observed. However, mutations will still be observed.
```cp
%-- Variable Reassignment --%
%% line 2 %% let var init: bool = false;
function say[init](b: bool ?= init): void { print.(b); }
%                             ^ refers to the `init` from line 2
say.(); % prints `false`

set init = true; % reassigns `init` from line 2, but not captured variable on line 3
say.();          % still prints `false`
%   ^ reads from the value `false` in line 2
```
```cp
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
```cp
%-- Variable Mutation --%
let a: mut int[] = List.<int>([42]);
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
-	::= "(" ","? ParameterType<?Named># ","?    ")" "=>" "void";
+TypeFunction
+	::= "(" ","? ParametersType<-Named, +Named> ")" "=>" "void";

Type ::=
	| TypeUnion
-	| TypeFunction<-Named, +Named>
+	| TypeFunction
;

ExpressionFunction
-	::= "(" ","? ParameterFunction# ","? ")" ":" "void" StatementBlock<-Break>;
+	::= "(" ","? ParametersFunction      ")" ":" "void" StatementBlock<-Break>;

-ParameterType<Named>
-	::= <Named+>(IDENTIFIER ":") Type;
+ParameterType<Named, Optional>
+	::= <Named+>(IDENTIFIER . <Optional->":") <Optional+>"?:" Type;

-ParameterFunction
-	::= (IDENTIFIER "=")? "var"? IDENTIFIER ":" Type;
+ParameterFunction<Optional>
+	::= (IDENTIFIER "=")? "var"? IDENTIFIER ":" Type . <Optional+>("?=" Expression);

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
	::= SemanticVariable? SemanticType;

SemanticParameter[unfixed: Boolean]
-	::= SemanticKey? SemanticVariable SemanticType;
+	::= SemanticKey? SemanticVariable SemanticType SemanticExpression?;
```

## Decorate
```diff
-Decorate(TypeFunction ::= "(" ","? ParameterType# ","? ")" "=>" "void") -> SemanticTypeFunction
-	:= (SemanticTypeFunction ...ParseList(ParameterType, SemanticParameterType));
+Decorate(TypeFunction ::= "(" ","? ParametersType ")" "=>" "void") -> SemanticTypeFunction
+	:= (SemanticTypeFunction ...Decorate(ParametersType));

Decorate(Type ::= TypeFunction) -> SemanticTypeFunction
	:= Decorate(TypeFunction);

-Decorate(ExpressionFunction ::= "(" ","? ParameterFunction# ","? ")" ":" "void" StatementBlock<-Break>) -> SemanticFunction
+Decorate(ExpressionFunction ::= "(" ","? ParametersFunction      ")" ":" "void" StatementBlock<-Break>) -> SemanticFunction
	:= (SemanticFunction
-		...ParseList(ParameterFunction, SemanticParameter)
+		...Decorate(ParametersFunction)
		Decorate(StatementBlock<-Break>)
	);

Decorate(Expression_Dynamic ::= ExpressionFunction) -> SemanticFunction
	:= Decorate(ExpressionFunction);

Decorate(ParameterType ::= Type) -> SemanticParameterType
-	:= (SemanticParameterType Decorate(Type));
+	:= (SemanticParameterType[optional=false] Decorate(Type));
+Decorate(ParameterType_Optional ::= "?:" Type) -> SemanticParameterType
+	:= (SemanticParameterType[optional=true] Decorate(Type));
Decorate(ParameterType_Named ::= IDENTIFIER ":" Type) -> SemanticParameterType
-	:= (SemanticParameterType
+	:= (SemanticParameterType[optional=false]
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
+Decorate(ParameterType_Named_Optional ::= IDENTIFIER "?:" Type) -> SemanticParameterType
+	:= (SemanticParameterType[optional=true]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);

Decorate(ParameterFunction ::= IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
Decorate(ParameterFunction ::= "var" IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
Decorate(ParameterFunction ::= IDENTIFIER__0 "=" IDENTIFIER__1 ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		(SemanticKey[id=TokenWorth(IDENTIFIER__0)])
		(SemanticVariable[id=TokenWorth(IDENTIFIER__1)])
		Decorate(Type)
	);
Decorate(ParameterFunction ::= IDENTIFIER__0 "=" "var" IDENTIFIER__1 ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		(SemanticKey[id=TokenWorth(IDENTIFIER__0)])
		(SemanticVariable[id=TokenWorth(IDENTIFIER__1)])
		Decorate(Type)
	);
+Decorate(ParameterFunction<+Optional> ::= IDENTIFIER ":" Type "?=" Expression) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression)
+	);
+Decorate(ParameterFunction<+Optional> ::= "var" IDENTIFIER ":" Type "?=" Expression) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression)
+	);
+Decorate(ParameterFunction<+Optional> ::= IDENTIFIER__0 "=" IDENTIFIER__1 ":" Type "?=" Expression) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticKey[id=TokenWorth(IDENTIFIER__0)])
+		(SemanticVariable[id=TokenWorth(IDENTIFIER__1)])
+		Decorate(Type)
+		Decorate(Expression)
+	);
+Decorate(ParameterFunction<+Optional> ::= IDENTIFIER__0 "=" "var" IDENTIFIER__1 ":" Type "?=" Expression) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticKey[id=TokenWorth(IDENTIFIER__0)])
+		(SemanticVariable[id=TokenWorth(IDENTIFIER__1)])
+		Decorate(Type)
+		Decorate(Expression)
+	);

+Decorate(ParametersType<Named> ::= ParameterType<?Named><-Optional># ","?) -> Sequence<SemanticParameterType>
+	:= Decorate(ParameterType<?Named><-Optional>#)
+Decorate(ParametersType<Named> ::= ParameterType<?Named><+Optional># ","?) -> Sequence<SemanticParameterType>
+	:= Decorate(ParameterType<?Named><+Optional>#)
+Decorate(ParametersType<Named> ::= ParameterType<?Named><-Optional># "," ParameterType<?Named><+Optional># ","?) -> Sequence<SemanticParameterType>
+	:= [
+		...Decorate(ParameterType<?Named><-Optional>#),
+		...Decorate(ParameterType<?Named><+Optional>#),
+	];

	Decorate(ParameterType# ::= ParameterType) -> Tuple<SemanticParameterType>
		:= [Decorate(ParameterType)];
	Decorate(ParameterType# ::= ParameterType# "," ParameterType) -> Sequence<SemanticParameterType>
		:= [
			...Decorate(ParameterType#),
			Decorate(ParameterType),
		];

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
-FunctionTypeOf(ExpressionFunction ::= "(" ","? ParameterFunction# ","? ")" ":" "void" StatementBlock<-Break>) -> SemanticTypeFunction
-	:= (SemanticTypeFunction ...FunctionTypeOf(ParameterFunction#));
+FunctionTypeOf(ExpressionFunction ::= "(" ","? ParametersFunction ")" ":" "void" StatementBlock<-Break>) -> SemanticTypeFunction
+	:= (SemanticTypeFunction ...FunctionTypeOf(ParametersFunction));

FunctionTypeOf(ParameterFunction ::= "var"? IDENTIFIER ":" Type) -> SemanticParameterType
	:= (SemanticParameterType
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
FunctionTypeOf(ParameterFunction ::= IDENTIFIER__0 "=" "var"? IDENTIFIER__1 ":" Type) -> SemanticParameterType
	:= (SemanticParameterType
		(SemanticVariable[id=TokenWorth(IDENTIFIER__0)])
		Decorate(Type)
	);

+FunctionTypeOf(ParametersFunction ::= ParameterFunction<-Optional># ","?) -> Sequence<SemanticParameter>
+	:= FunctionTypeOf(ParameterFunction<-Optional>#);
+FunctionTypeOf(ParametersFunction ::= ParameterFunction<+Optional># ","?) -> Sequence<SemanticParameter>
+	:= FunctionTypeOf(ParameterFunction<+Optional>#);
+FunctionTypeOf(ParametersFunction ::= ParameterFunction<-Optional># "," ParameterFunction<+Optional># ","?) -> Sequence<SemanticParameter>
+	:= [
+		...FunctionTypeOf(ParameterFunction<-Optional>#),
+		...FunctionTypeOf(ParameterFunction<+Optional>#),
+	];

	FunctionTypeOf(ParameterFunction# ::= ParameterFunction) -> Tuple<SemanticParameterType>
		:= [FunctionTypeOf(ParameterFunction)];
	FunctionTypeOf(ParameterFunction# ::= ParameterFunction# "," ParameterFunction) -> Sequence<SemanticParameterType>
		:= [
			...FunctionTypeOf(ParameterFunction#),
			FunctionTypeOf(ParameterFunction),
		];
```
