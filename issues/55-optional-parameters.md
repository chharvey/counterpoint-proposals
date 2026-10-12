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
type Meal = \(?:str, str) => void;                        %> ParseError
```
Specifying an initializer that mismatches the parameter type results in a TypeError.
```cpl
func moveForward(steps?: int = false): void {;} %> TypeError: `false` not assignable to `int`
```

Optional parameters may be writable (reassignable within the function):
```cpl
func greet(mut greeting?: str = "Hello"): void {
	greeting = if greeting == "" then "Hi" else greeting;
	"""{{ greeting }}, world!""";
}
greet; %: \(?: str) => void
```

Named arguments give us a fantastic advantage here:
They let us skip optional parameters so that we don’t have to remember their default values.
Say we want to call `move2D` with only a `player` and a `y`, but no `x`.
Since unnamed arguments must be provided in order, we need to provide the argument for `x` before the argument for `y`.
That would require us knowing its default value, which introduces unnecessary redundancy and upkeep.
But with named arguments, we can skip optional arguments in the function call.
```cpl
func move2D_positional(x?: float = 0.0, y?: float = 0.0): void { return; }

move2D(0.0, 1.0);
%       ^ need to write default value of `x`

func move2D_named($x?: float = 0.0, $y?: float = 0.0): void { return; }

move2D(y= 1.0); % skip `x`, using its default value
```

## Default Parameter Evaluation
Optional parameter initializers are evaluated when the function is *called*, not when it’s *defined*.
```cpl
func say_hello(): void {
	print("hello");
};
func run(x?: void = say_hello()): void {}; % does not print "hello" here
run(); % prints "hello" here
run(); % prints "hello" again
```

If an optional parameter initializer references a variable, it must be captured, and it refers to the variable bound to the environment in which it’s *initialized*, not in which the function is *called*. And, if that variable is ever reassigned outside the function, the reassignment is not observed. However, mutations will still be observed.
```cpl
%-- Variable Reassignment --%
%% line 2 %% val mut init: bool = false;
func say(b?: bool = init) with (init): void { print(b); }
%                   ^ refers to the `init` from line 2
say(); % prints `false`

set init = true; % reassigns `init` from line 2, but not captured variable on line 3
say();           % still prints `false`
%   ^ reads from the value `false` in line 2

%% line 11 %% val mut init2: bool = false;
func say2(b?: bool = init2) with (ref init2): void { print(b); }
%                    ^ refers to `init2` from line 11
say2(); % prints `false`
set init2 = true; % reassigns `init` from line 2, and ref-captured variable on line 12
say2(); % prints `true`
%   ^ reads from the value `true` in line 15
```
```cpl
%-- Import Shadowing --%
% Module "a"
%% line 3 %% val init: bool = false;
public func say(b?: bool = init) with (init): void { print(b); }
%                          ^ refers to the `init` from line 3
say(); % prints `false`

% Module "b"
from "a" import say;
val init: bool = true;
say();                % still prints `false`
%  ^ reads from same `init` as Module "a" (not new `init` from Module "b")
```
```cpl
%-- Variable Mutation --%
val a: mut [int] = [42];
func twice(x?: int = a.get(0)) with (a): int => x * 2;
twice(); %== 84
a.set(0, 12);
twice(); %== 24
```

# Specification

## Syntax
```diff
ParametersType ::=
-	| ","? EntryType<-Named><-Optional># ("," EntryType<+Named><-Optional>#)? ","?
-	| ","?                                    EntryType<+Named><-Optional>#   ","?
+	| ","? EntryType<-Named><-Optional># ("," EntryType<-Named><+Optional>#)? ("," EntryType<+Named><∓Optional>#)? ","?
+	| ","?                                    EntryType<-Named><+Optional>#   ("," EntryType<+Named><∓Optional>#)? ","?
+	| ","?                                                                         EntryType<+Named><∓Optional>#   ","?
;

-ParameterFunction<Named>           ::= (
+ParameterFunction<Named, Optional> ::= (
	| <Named+>(Word "=") ("_" | "mut"? IDENTIFIER)
	| <Named+>("mut"? "$" IDENTIFIER)
-)                ":" Type;
+) <Optional+>"?" ":" Type & <Optional+>("=" Expression<+Block><-Break><-Return>);

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
SemanticParameterFunction[named: true][writable: Boolean][id: Nil]
-	::= SemanticKey SemanticType;
+	::= SemanticKey SemanticType SemanticExpression?;
SemanticParameterFunction[named: true][writable: Boolean][id: RealNumber]
-	::= SemanticKey? SemanticType;
+	::= SemanticKey? SemanticType SemanticExpression?;
SemanticParameterFunction[named: false][writable: Boolean][id?: RealNumber]
-	::= SemanticType;
+	::= SemanticType SemanticExpression?;
```

## Decorate
```diff
Decorate(ParametersType ::= ","? EntryType<-Named><-Optional># ","?) -> Sequence<SemanticItemType>
	:= ParseList(EntryType<-Named><-Optional>, SemanticItemType);
-Decorate(ParametersType ::= ","? EntryType<+Named><-Optional># ","?) -> Sequence<SemanticPropertyType>
-	:= ParseList(EntryType<+Named><-Optional>, SemanticPropertyType);
-Decorate(ParametersType ::= ","? EntryType<-Named><-Optional># "," EntryType<+Named><-Optional># ","?) -> Sequence<...Sequence<SemanticItemType>, ...Sequence<SemanticPropertyType>>
-	:= [
-		...ParseList(EntryType<-Named><-Optional>, SemanticItemType),
-		...ParseList(EntryType<+Named><-Optional>, SemanticPropertyType),
-	];
+Decorate(ParametersType ::= ","? EntryType<-Named><+Optional># ","?) -> Sequence<SemanticItemType>
+	:= ParseList(EntryType<-Named><+Optional>, SemanticItemType);
+Decorate(ParametersType ::= ","? EntryType<+Named><∓Optional># ","?) -> Sequence<SemanticPropertyType>
+	:= ParseList(EntryType<+Named><∓Optional>, SemanticPropertyType);
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

-Decorate(ParameterFunction<-Named>            ::= "_" ":" Type) -> SemanticParameterFunction
+Decorate(ParameterFunction<-Named><-Optional> ::= "_" ":" Type) -> SemanticParameterFunction
	:= (SemanticParameterFunction[named=false][writable=false][id=*nil*]
		Decorate(Type)
	);
+Decorate(ParameterFunction<-Named><+Optional> ::= "_" "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameterFunction
+	:= (SemanticParameterFunction[named=false][writable=false][id=*nil*]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
-Decorate(ParameterFunction<-Named>            ::= IDENTIFIER ":" Type) -> SemanticParameterFunction
+Decorate(ParameterFunction<-Named><-Optional> ::= IDENTIFIER ":" Type) -> SemanticParameterFunction
	:= (SemanticParameterFunction[named=false][writable=false][id=TokenWorth(IDENTIFIER)]
		Decorate(Type)
	);
+Decorate(ParameterFunction<-Named><+Optional> ::= IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameterFunction
+	:= (SemanticParameterFunction[named=false][writable=false][id=TokenWorth(IDENTIFIER)]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
-Decorate(ParameterFunction<-Named>            ::= "mut" IDENTIFIER ":" Type) -> SemanticParameterFunction
+Decorate(ParameterFunction<-Named><-Optional> ::= "mut" IDENTIFIER ":" Type) -> SemanticParameterFunction
	:= (SemanticParameterFunction[named=false][writable=true][id=TokenWorth(IDENTIFIER)]
		Decorate(Type)
	);
+Decorate(ParameterFunction<-Named><+Optional> ::= "mut" IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameterFunction
+	:= (SemanticParameterFunction[named=false][writable=true][id=TokenWorth(IDENTIFIER)]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
-Decorate(ParameterFunction<+Named>            ::= Word "=" "_" ":" Type) -> SemanticParameterFunction
+Decorate(ParameterFunction<+Named><-Optional> ::= Word "=" "_" ":" Type) -> SemanticParameterFunction
	:= (SemanticParameterFunction[named=true][writable=false][id=*nil*]
		Decorate(Word)
		Decorate(Type)
	);
+Decorate(ParameterFunction<+Named><+Optional> ::= Word "=" "_" "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameterFunction
+	:= (SemanticParameterFunction[named=true][writable=false][id=*nil*]
+		Decorate(Word)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
-Decorate(ParameterFunction<+Named>            ::= Word "=" IDENTIFIER ":" Type) -> SemanticParameterFunction
+Decorate(ParameterFunction<+Named><-Optional> ::= Word "=" IDENTIFIER ":" Type) -> SemanticParameterFunction
	:= (SemanticParameterFunction[named=true][writable=false][id=TokenWorth(IDENTIFIER)]
		Decorate(Word)
		Decorate(Type)
	);
+Decorate(ParameterFunction<+Named><+Optional> ::= Word "=" IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameterFunction
+	:= (SemanticParameterFunction[named=true][writable=false][id=TokenWorth(IDENTIFIER)]
+		Decorate(Word)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
-Decorate(ParameterFunction<+Named>            ::= Word "=" "mut" IDENTIFIER ":" Type) -> SemanticParameterFunction
+Decorate(ParameterFunction<+Named><-Optional> ::= Word "=" "mut" IDENTIFIER ":" Type) -> SemanticParameterFunction
	:= (SemanticParameterFunction[named=true][writable=true][id=TokenWorth(IDENTIFIER)]
		Decorate(Word)
		Decorate(Type)
	);
+Decorate(ParameterFunction<+Named><+Optional> ::= Word "=" "mut" IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameterFunction
+	:= (SemanticParameterFunction[named=true][writable=true][id=TokenWorth(IDENTIFIER)]
+		Decorate(Word)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
-Decorate(ParameterFunction<+Named>            ::= "$" IDENTIFIER ":" Type) -> SemanticParameterFunction
+Decorate(ParameterFunction<+Named><-Optional> ::= "$" IDENTIFIER ":" Type) -> SemanticParameterFunction
	:= (SemanticParameterFunction[named=true][writable=false][id=TokenWorth(IDENTIFIER)]
		Decorate(Type)
	);
+Decorate(ParameterFunction<+Named><+Optional> ::= "$" IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameterFunction
+	:= (SemanticParameterFunction[named=true][writable=false][id=TokenWorth(IDENTIFIER)]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);
-Decorate(ParameterFunction<+Named>            ::= "mut" "$" IDENTIFIER ":" Type) -> SemanticParameterFunction
+Decorate(ParameterFunction<+Named><-Optional> ::= "mut" "$" IDENTIFIER ":" Type) -> SemanticParameterFunction
	:= (SemanticParameterFunction[named=true][writable=true][id=TokenWorth(IDENTIFIER)]
		Decorate(Type)
	);
+Decorate(ParameterFunction<+Named><+Optional> ::= "mut" "$" IDENTIFIER "?" ":" Type "=" Expression<+Block><-Break><-Return>) -> SemanticParameterFunction
+	:= (SemanticParameterFunction[named=true][writable=true][id=TokenWorth(IDENTIFIER)]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><-Return>)
+	);

-Decorate(ParametersFunction ::= ","? ParameterFunction<-Named># ","?) -> Sequence<SemanticParameterFunction>
-	:= ParseList(ParameterFunction<-Named>, SemanticParameterFunction);
-Decorate(ParametersFunction ::= ","? ParameterFunction<+Named># ","?) -> Sequence<SemanticParameterFunction>
-	:= ParseList(ParameterFunction<+Named>, SemanticParameterFunction);
-Decorate(ParametersFunction ::= ","? ParameterFunction<-Named># "," ParameterFunction<+Named># ","?) -> Sequence<SemanticParameterFunction>
-	:= [
-		...ParseList(ParameterFunction<-Named>, SemanticParameterFunction),
-		...ParseList(ParameterFunction<+Named>, SemanticParameterFunction),
-	];
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># ","?) -> Sequence<SemanticParameterFunction>
+	:= ParseList(ParameterFunction<-Named><-Optional>, SemanticParameterFunction);
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><+Optional># ","?) -> Sequence<SemanticParameterFunction>
+	:= ParseList(ParameterFunction<-Named><+Optional>, SemanticParameterFunction);
+Decorate(ParametersFunction ::= ","? ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticParameterFunction>
+	:= ParseList(ParameterFunction<+Named><∓Optional>, SemanticParameterFunction);
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># "," ParameterFunction<-Named><+Optional># ","?) -> Sequence<SemanticParameterFunction>
+	:= [
+		...ParseList(ParameterFunction<-Named><-Optional>, SemanticParameterFunction),
+		...ParseList(ParameterFunction<-Named><+Optional>, SemanticParameterFunction),
+	];
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># "," ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticParameterFunction>
+	:= [
+		...ParseList(ParameterFunction<-Named><-Optional>, SemanticParameterFunction),
+		...ParseList(ParameterFunction<+Named><∓Optional>, SemanticParameterFunction),
+	];
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><+Optional># "," ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticParameterFunction>
+	:= [
+		...ParseList(ParameterFunction<-Named><+Optional>, SemanticParameterFunction),
+		...ParseList(ParameterFunction<+Named><∓Optional>, SemanticParameterFunction),
+	];
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named><-Optional># "," ParameterFunction<-Named><+Optional># "," ParameterFunction<+Named><∓Optional># ","?) -> Sequence<SemanticParameterFunction>
+	:= [
+		...ParseList(ParameterFunction<-Named><-Optional>, SemanticParameterFunction),
+		...ParseList(ParameterFunction<-Named><+Optional>, SemanticParameterFunction),
+		...ParseList(ParameterFunction<+Named><∓Optional>, SemanticParameterFunction),
+	];
```
