**Optional parameters** allow arguments to be omitted from a function call.

# Discussion
An optional parameter must have a **default value**, and when the argument for that optional parameter is omitted, the function is called as if the default value were provided.

```cpl
func moveForward(steps?: int = 1): void {
	steps; %: int
}
moveForward; %: \(?: int) => void
```
The code `= 1` is called an **initializer** and defines the default value of the `steps` parameter. It’s similar to the initializer of a variable declaration. When `moveForward` is called without that argument, the default value of `1` is assumed.

Notice the type signature’s new syntax: `\(?: int) => void`. This means that the function may be called with 0 or 1 argument, and if it is called with 1 argument, that argument *must* be of type `int`. Function calls are not covered in this version.

The syntax of optional type parameters carries over into unnamed parameters of a type signature.
```cpl
type BinaryOperatorTypeUnnamed = \(float, ?:float) => void;
```
The signature above indicates that a function of type `BinaryOperatorTypeUnnamed` has 1 required parameter and 1 optional parameter. Optional parameters’ default values are an implemenation detail, so they cannot be specified in type signatures. And since the parameters of this type signature are unnamed, an implementing function can only be called with positional arguments.

All optional parameters must be declared after all required parameters; otherwise it’s a syntax error.
```cpl
func breakfast(entree: str, dessert?: str = ""): void {;} % fine, type `\(str, ?: str) => void`
func dinner   (dessert?: str = "", entree: str): void {;} %> ParseError
type Meal = \(?:str, str) => void;                            %> ParseError
```
Specifying an initializer that mismatches the parameter type results in a TypeError.
```cpl
func moveForward(steps?: int = false): void {;} %> TypeError: `false` not assignable to `int`
```

Optional parameters may be unfixed (reassignable within the function):
```cpl
func greet(mut greeting?: str = "Hello"): void {
	greeting = if greeting == "" then "Hi" else greeting;
	"""{{ greeting }}, world!""";
}
greet; %: \(?: str) => void
```

## Default Parameter Evaluation
Optional parameter initializers are evaluated when the function is *called*, not when it’s *defined*.
```cpl
func say_hello(): void {
	print.("hello");
};
func run[say_hello](x?: void = say_hello.()): void {}; % does not print "hello" here
run.();                                                % prints "hello" here
run.();                                                % prints "hello" again
```

If an optional parameter initializer references a variable, it must be captured (as shown above), and it refers to the variable bound to the environment in which it’s *initialized*, not in which the function is *called*. And, if that variable is ever reassigned outside the function, the reassignment is not observed. However, mutations will still be observed.
```cpl
%-- Variable Reassignment --%
%% line 2 %% let mut init: bool = false;
func say[init](b?: bool = init): void { print.(b); }
%                         ^ refers to the `init` from line 2
say.(); % prints `false`

set init = true; % reassigns `init` from line 2, but not captured variable on line 3
say.();          % still prints `false`
%   ^ reads from the value `false` in line 2
```
```cpl
%-- Import Shadowing --%
% Module "a"
%% line 3 %% let init: bool = false;
public func say[init](b?: bool = init): void { print.(b); }
%                                ^ refers to the `init` from line 3
say.(); % prints `false`

% Module "b"
from "a" import say;
let init: bool = true;
say.();                % still prints `false`
%   ^ reads from same `init` as Module "a" (not new `init` from Module "b")
```
```cpl
%-- Variable Mutation --%
let a: mut [int] = [42];
func twice[a](x?: int = a.[0]): int => x * 2;
twice.(); %== 84
set a.[0] = 12;
twice.(); %== 24
```

# Specification

## Syntax
```diff
TypeFunction
	::= "\" "(" ParametersType? ")" "=>" ("void" | Type);

DeclaredFunction   ::=     "(" ParametersFunction? ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression<+Block><-Break><+Return> ";");
ExpressionFunction ::= "\" "(" ParametersFunction? ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression<+Block><-Break><+Return>);

-ParameterFunction<Named>
+ParameterFunction<Named, Optional>
-	::= <Named->"mut"? <Named+>(Word "=" "mut"? | "mut"? "$") ("_" | IDENTIFIER)                ":" Type;
+	::= <Named->"mut"? <Named+>(Word "=" "mut"? | "mut"? "$") ("_" | IDENTIFIER) <Optional+>"?" ":" Type & <Optional+>("=" Expression<+Block><-Break><-Return>);

ParametersType ::=
-	| ","? EntryType<-Named><-Optional># ("," EntryType<+Named><-Optional>#)? ","?
-	| ","?                                    EntryType<+Named><-Optional>#   ","?
+	| ","? EntryType<-Named><-Optional># ("," EntryType<-Named><+Optional>#)? ("," EntryType<+Named><∓Optional>#)? ","?
+	| ","?                                    EntryType<-Named><+Optional>#   ("," EntryType<+Named><∓Optional>#)? ","?
+	| ","?                                                                         EntryType<+Named><∓Optional>#   ","?
;

ParametersFunction ::=
-	| ","? ParameterFunction<-Named># ("," ParameterFunction<+Named>#)? ","?
-	| ","?                                 ParameterFunction<+Named>#   ","?
+	| ","? ParameterFunction<-Named><-Optional># ("," ParameterFunction<-Named><+Optional>#)? ("," ParameterFunction<+Named><∓Optional>#)? ","?
+	| ","?                                            ParameterFunction<-Named><+Optional>#   ("," ParameterFunction<+Named><∓Optional>#)? ","?
+	| ","?                                                                                         ParameterFunction<+Named><∓Optional>#   ","?
;
```

## Semantics
```diff
SemanticParameter[unfixed: Boolean]
-	::= SemanticKey? SemanticVariable SemanticType;
+	::= SemanticKey? SemanticVariable SemanticType SemanticExpression?;
```

## Decorate
```diff
-Decorate(ParameterFunction<-Named>            ::= "_" ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Named><-Optional> ::= "_" ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		Decorate(Type)
	);
-Decorate(ParameterFunction<-Named>            ::= IDENTIFIER ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Named><-Optional> ::= IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
-Decorate(ParameterFunction<-Named>            ::= "mut" "_" ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Named><-Optional> ::= "mut" "_" ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		Decorate(Type)
	);
-Decorate(ParameterFunction<-Named>            ::= "mut" IDENTIFIER ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<-Named><-Optional> ::= "mut" IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
-Decorate(ParameterFunction<+Named>            ::= "$" "_" ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<+Named><-Optional> ::= "$" "_" ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		(SemanticKey[id=TokenWorth("_")])
		Decorate(Type)
	);
-Decorate(ParameterFunction<+Named>            ::= "$" IDENTIFIER ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<+Named><-Optional> ::= "$" IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		(SemanticKey[id=TokenWorth(IDENTIFIER)])
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
-Decorate(ParameterFunction<+Named>            ::= "mut" "$" "_" ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<+Named><-Optional> ::= "mut" "$" "_" ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		(SemanticKey[id=TokenWorth("_")])
		Decorate(Type)
	);
-Decorate(ParameterFunction<+Named>            ::= "mut" "$" IDENTIFIER ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<+Named><-Optional> ::= "mut" "$" IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		(SemanticKey[id=TokenWorth(IDENTIFIER)])
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
-Decorate(ParameterFunction<+Named>            ::= Word "=" "_" ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<+Named><-Optional> ::= Word "=" "_" ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		Decorate(Word)
		Decorate(Type)
	);
-Decorate(ParameterFunction<+Named>            ::= Word "=" IDENTIFIER ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<+Named><-Optional> ::= Word "=" IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=false]
		Decorate(Word)
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
-Decorate(ParameterFunction<+Named>            ::= Word "=" "mut" "_" ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<+Named><-Optional> ::= Word "=" "mut" "_" ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		Decorate(Word)
		Decorate(Type)
	);
-Decorate(ParameterFunction<+Named>            ::= Word "=" "mut" IDENTIFIER ":" Type) -> SemanticParameter
+Decorate(ParameterFunction<+Named><-Optional> ::= Word "=" "mut" IDENTIFIER ":" Type) -> SemanticParameter
	:= (SemanticParameter[unfixed=true]
		Decorate(Word)
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		Decorate(Type)
	);
+Decorate(ParameterFunction<-Named><+Optional> ::= "_" "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<-Named><+Optional> ::= IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<-Named><+Optional> ::= "mut" "_" "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<-Named><+Optional> ::= "mut" IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Named><+Optional> ::= "$" "_" "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticKey[id=TokenWorth("_")])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Named><+Optional> ::= "$" IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticKey[id=TokenWorth(IDENTIFIER)])
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Named><+Optional> ::= "mut" "$" "_" "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticKey[id=TokenWorth("_")])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Named><+Optional> ::= "mut" "$" IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticKey[id=TokenWorth(IDENTIFIER)])
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Named><+Optional> ::= Word "=" "_" "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		Decorate(Word)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Named><+Optional> ::= Word "=" IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		Decorate(Word)
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Named><+Optional> ::= Word "=" "mut" "_" "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		Decorate(Word)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
+Decorate(ParameterFunction<+Named><+Optional> ::= Word "=" "mut" IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		Decorate(Word)
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);

Decorate(ParametersType ::= ","? EntryType<-Named><-Optional># ","?) -> Sequence<SemanticItemType>
	:= ParseList(EntryType<-Named><-Optional>, SemanticItemType)
-Decorate(ParametersType ::= ","? EntryType<+Named><-Optional># ","?) -> Sequence<SemanticPropertyType>
-	:= ParseList(EntryType<+Named><-Optional>, SemanticPropertyType)
-Decorate(ParametersType ::= ","? EntryType<-Named><-Optional># "," EntryType<+Named><-Optional># ","?) -> Sequence<...Sequence<SemanticItemType>, ...Sequence<SemanticPropertyType>>
-	:= [
-		...ParseList(EntryType<-Named><-Optional>, SemanticItemType),
-		...ParseList(EntryType<+Named><-Optional>, SemanticPropertyType),
-	];
+Decorate(ParametersType ::= ","? EntryType<-Named><+Optional># ","?) -> Sequence<SemanticItemType>
+	:= ParseList(EntryType<-Named><+Optional>, SemanticItemType)
+Decorate(ParametersType ::= ","? EntryType<+Named><∓Optional># ","?) -> Sequence<SemanticPropertyType>
+	:= ParseList(EntryType<+Named><∓Optional>, SemanticPropertyType)
+Decorate(ParametersType ::= ","? EntryType<-Named><-Optional># "," EntryType<-Named><+Optional># ","?) -> Sequence<SemanticItemType>
+	:= [
+		...ParseList(EntryType<-Named><-Optional>, SemanticItemType),
+		...ParseList(EntryType<-Named><+Optional>, SemanticItemType),
+	];
+Decorate(ParametersType ::= ","? EntryType<-Named><-Optional># "," EntryType<+Named><∓Optional># ","?) -> Sequence<...Sequence<SemanticItemType>, ...Sequence<SemanticPropertyType>>
+	:= [
+		...ParseList(EntryType<-Named><-Optional>, SemanticItemType),
+		...ParseList(EntryType<+Named><∓Optional>, SemanticPropertyType),
+	];
+Decorate(ParametersType ::= ","? EntryType<-Named><+Optional># "," EntryType<+Named><∓Optional># ","?) -> Sequence<...Sequence<SemanticItemType>, ...Sequence<SemanticPropertyType>>
+	:= [
+		...ParseList(EntryType<-Named><+Optional>, SemanticItemType),
+		...ParseList(EntryType<+Named><∓Optional>, SemanticPropertyType),
+	];
+Decorate(ParametersType ::= ","? EntryType<-Named><-Optional># "," EntryType<-Named><+Optional># "," EntryType<+Named><∓Optional># ","?) -> Sequence<...Sequence<SemanticItemType>, ...Sequence<SemanticPropertyType>>
+	:= [
+		...ParseList(EntryType<-Named><-Optional>, SemanticItemType),
+		...ParseList(EntryType<-Named><+Optional>, SemanticItemType),
+		...ParseList(EntryType<+Named><∓Optional>, SemanticPropertyType),
+	];

-Decorate(ParametersFunction ::= ","? ParameterFunction<-Named># ","?) -> Sequence<SemanticParameter>
-	:= ParseList(ParameterFunction<-Named>, SemanticParameter)
-Decorate(ParametersFunction ::= ","? ParameterFunction<+Named># ","?) -> Sequence<SemanticParameter>
-	:= ParseList(ParameterFunction<+Named>, SemanticParameter)
-Decorate(ParametersFunction ::= ","? ParameterFunction<-Named># "," ParameterFunction<+Named># ","?) -> Sequence<SemanticParameter>
-	:= [
-		...ParseList(ParameterFunction<-Named>, SemanticParameter),
-		...ParseList(ParameterFunction<+Named>, SemanticParameter),
-	];

+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># ","?) -> Sequence<SemanticParameter>
+	:= ParseList(ParameterFunction<-Named><-Optional>, SemanticParameter)
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><+Optional># ","?) -> Sequence<SemanticParameter>
+	:= ParseList(ParameterFunction<-Named><+Optional>, SemanticParameter)
+Decorate(ParametersFunction ::= ","? ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticParameter>
+	:= ParseList(ParameterFunction<+Named><∓Optional>, SemanticParameter)
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># "," ParameterFunction<-Named><+Optional># ","?) -> Sequence<SemanticParameter>
+	:= [
+		...ParseList(ParameterFunction<-Named><-Optional>, SemanticParameter),
+		...ParseList(ParameterFunction<-Named><+Optional>, SemanticParameter),
+	];
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># "," ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticParameter>
+	:= [
+		...ParseList(ParameterFunction<-Named><-Optional>, SemanticParameter),
+		...ParseList(ParameterFunction<+Named><∓Optional>, SemanticParameter),
+	];
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><+Optional># "," ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticParameter>
+	:= [
+		...ParseList(ParameterFunction<-Named><+Optional>, SemanticParameter),
+		...ParseList(ParameterFunction<+Named><∓Optional>, SemanticParameter),
+	];
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># "," ParameterFunction<-Named><+Optional># "," ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticParameter>
+	:= [
+		...ParseList(ParameterFunction<-Named><-Optional>, SemanticParameter),
+		...ParseList(ParameterFunction<-Named><+Optional>, SemanticParameter),
+		...ParseList(ParameterFunction<+Named><∓Optional>, SemanticParameter),
+	];
```

## FunctionTypeOf
```diff
-	FunctionTypeOf(ParametersFunction ::= ParameterFunction<-Named># ","?) -> Sequence<SemanticItemType>
-		:= FunctionTypeOf(ParameterFunction<-Named>#);
-	FunctionTypeOf(ParametersFunction ::= ParameterFunction<+Named># ","?) -> Sequence<SemanticPropertyType>
-		:= FunctionTypeOf(ParameterFunction<+Named>#);
-	FunctionTypeOf(ParametersFunction ::= ParameterFunction<-Named># "," ParameterFunction<+Named># ","?) -> Sequence<SemanticItemType | SemanticPropertyType>
-		:= [
-			...FunctionTypeOf(ParameterFunction<-Named>#),
-			...FunctionTypeOf(ParameterFunction<+Named>#),
-		];
+	FunctionTypeOf(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># ","?) -> Sequence<SemanticItemType>
+		:= FunctionTypeOf(ParameterFunction<-Named><-Optional>#)
+	FunctionTypeOf(ParametersFunction ::= ","? ParameterFunction<-Named><+Optional># ","?) -> Sequence<SemanticItemType>
+		:= FunctionTypeOf(ParameterFunction<-Named><+Optional>#)
+	FunctionTypeOf(ParametersFunction ::= ","? ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticPropertyType>
+		:= FunctionTypeOf(ParameterFunction<+Named><∓Optional>#)
+	FunctionTypeOf(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># "," ParameterFunction<-Named><+Optional># ","?) -> Sequence<SemanticItemType>
+		:= [
+			...FunctionTypeOf(ParameterFunction<-Named><+Optional>#),
+			...FunctionTypeOf(ParameterFunction<-Named><-Optional>#),
+		];
+	FunctionTypeOf(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># "," ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticItemType | SemanticPropertyType>
+		:= [
+			...FunctionTypeOf(ParameterFunction<+Named><∓Optional>#),
+			...FunctionTypeOf(ParameterFunction<-Named><-Optional>#),
+		];
+	FunctionTypeOf(ParametersFunction ::= ","? ParameterFunction<-Named><+Optional># "," ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticItemType | SemanticPropertyType>
+		:= [
+			...FunctionTypeOf(ParameterFunction<+Named><∓Optional>#),
+			...FunctionTypeOf(ParameterFunction<-Named><+Optional>#),
+		];
+	FunctionTypeOf(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># "," ParameterFunction<-Named><+Optional># "," ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticItemType | SemanticPropertyType>
+		:= [
+			...FunctionTypeOf(ParameterFunction<-Named><+Optional>#),
+			...FunctionTypeOf(ParameterFunction<-Named><-Optional>#),
+			...FunctionTypeOf(ParameterFunction<+Named><∓Optional>#),
+		];

-	FunctionTypeOf(ParameterFunction<Named>#           ::= ParameterFunction<?Named>)            -> Tuple<SemanticItemType | SemanticPropertyType>
+	FunctionTypeOf(ParameterFunction<Named, Optional># ::= ParameterFunction<?Named><?Optional>) -> Tuple<SemanticItemType | SemanticPropertyType>
-		:= [FunctionTypeOf(ParameterFunction<?Named>)];
+		:= [FunctionTypeOf(ParameterFunction<?Named><?Optional>)];
-	FunctionTypeOf(ParameterFunction<Named>#           ::= ParameterFunction<?Named>#            "," ParameterFunction<?Named>)            -> Sequence<SemanticItemType | SemanticPropertyType>
+	FunctionTypeOf(ParameterFunction<Named, Optional># ::= ParameterFunction<?Named><?Optional># "," ParameterFunction<?Named><?Optional>) -> Sequence<SemanticItemType | SemanticPropertyType>
-		:= [
-			...FunctionTypeOf(ParameterFunction<?Named>#),
-			FunctionTypeOf(ParameterFunction<?Named>),
+			...FunctionTypeOf(ParameterFunction<?Named><?Optional>#),
+			FunctionTypeOf(ParameterFunction<?Named><?Optional>),
-		];

-FunctionTypeOf(ParameterFunction<-Named>            ::= "mut"? ("_" | IDENTIFIER)                ":" Type)                                                        -> SemanticItemType
+FunctionTypeOf(ParameterFunction<-Named><∓Optional> ::= "mut"? ("_" | IDENTIFIER) <Optional+>"?" ":" Type & <Optional+>("=" Expression<+Block><-Break><-Return>)) -> SemanticItemType
	:= (SemanticItemType[optionanl=false] Decorate(Type));
-FunctionTypeOf(ParameterFunction<+Named>            ::= "mut"? "$" ("_" | IDENTIFIER)                ":" Type)                                                        -> SemanticPropertyType
+FunctionTypeOf(ParameterFunction<+Named><∓Optional> ::= "mut"? "$" ("_" | IDENTIFIER) <Optional+>"?" ":" Type & <Optional+>("=" Expression<+Block><-Break><-Return>)) -> SemanticPropertyType
	:= (SemanticPropertyType[optional=false]
		(SemanticKey[id=TokenWorth("_" | IDENTIFIER)])
		Decorate(Type)
	);
-FunctionTypeOf(ParameterFunction<+Named>            ::= Word "=" "mut"? ("_" | IDENTIFIER)                ":" Type)                                                        -> SemanticPropertyType
+FunctionTypeOf(ParameterFunction<+Named><∓Optional> ::= Word "=" "mut"? ("_" | IDENTIFIER) <Optional+>"?" ":" Type & <Optional+>("=" Expression<+Block><-Break><-Return>)) -> SemanticPropertyType
	:= (SemanticPropertyType[optional=false]
		Decorate(Word)
		Decorate(Type)
	);
```
