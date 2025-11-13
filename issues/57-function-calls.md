Syntax for calling functions.

# Discussion
This issue covers only the syntax and compile-time semantics of calling synchronous functions. All functions in this version are synchronous. Asynchronous and generator functions are in the next version.

A function call has the syntax `‹f›.(‹args›)`, where `‹f›` is a compound expression representing a function, and `‹args›` are zero or more arguments passed into the function.

Function arguments may be a mix of **positional arguments** and **named arguments**, but all positional arguments *must* come before all named arguments in a function call. Positional arguments are simply expressions, and named arguments are expressions followed by a label, akin to a record property: `label= (argument)`.

Positional arguments correspond to positional parameters, and named arguments correspond to named parameters. It’s an error to assign a named argument to a positional parameter or vice versa.
```cpl
function add(a: int, b: int): int => a + b;
function subtract(a: int, $b: int): int => a - b;
function multiply($a: int, $b: int): int => a * b;
function divide($a: int, b: int): int => a / b; %> Error % positional param cannot appear after named param
add.(5, 3);            %== 8
add.(5, b= 3);         %> Error
subtract.(5, 3);       %> Error
subtract.(5, b= 3);    %== 2
multiply.(a= 5, b= 3); %== 15
multiply.(b= 3, a= 5); %== 15
multiply.(5, b= 3);    %> Error
multiply.(3, 5);       %> Error
```

Arguments are assigned to parameters at compile-time. All positional arguments are assigned first, followed by all named arguments. Any doubly-asigned arguments are an error. Any required parameters left unassigned will result in a type error, as well as any unassigned arguments.
```cpl
add.(5, 3, 2);               %> TypeError: too many arguments
subtract.(5);                %> TypeError: too few arguments
subtract.(2, b= 3, a= 5);    %> TypeError: too many arguments
multiply.(b= 5, c= 3);       %> TypeError: argument for `a` is required
multiply.(a= 5, b= 3, c= 2); %> TypeError: too many arguments
```

After all arguments are assigned, and assuming no type errors, any unassigned optional parameters are then evaluated. This happens in the function *call*, not in its definition.
```cpl
function multiply(a: float, b: float ?= 1.0): float {
	return a * b;
};
multiply.(2.0); % evaluates default `b` here
multiply.(2.0); % evaluates default `b` here again
```

At runtime, arguments are always evaluated first, in left-to-right order, before they are sent into the function and the function is executed. The evaluation of optional parameters happens at the *call site*, but within the scope of the *definition site*. This means that any variables within an optional parameter’s default value are bound to the environment in which the function was defined. See #55 for details.

# Specification

## Syntax
```diff
Property <Break, Return> ::= Word                                "="  Expression<+Block><?Break><?Return>;
Case     <Break, Return> ::= Expression<+Block><?Break><?Return> "->" Expression<+Block><?Break><?Return>;

FunctionArguments<Break, Return> ::=
-	| "(" ","? Expression<+Block><?Break><?Return>#                                   ","? ")"
+	| "(" ","? Expression<+Block><?Break><?Return># ("," Property<?Break><?Return>#)? ","? ")"
+	| "(" ","?                                           Property<?Break><?Return>#   ","? ")"
;
```

## Semantic Schema
```diff
SemanticProperty
	::= SemanticKey SemanticExpression;

SemanticCall
-	::= SemanticExpression SemanticType* SemanticExpression*;
+	::= SemanticExpression SemanticType* SemanticExpression* SemanticProperty*;
```

## Decorate
```diff
-Decorate(FunctionArguments ::= "(" ( ","? Expression<+Block><?Break><?Return># ","? )? ")") -> Sequence<SemanticExpression>
-	:= ParseList(Expression<+Block><?Break><?Return>, SemanticExpression);
+Decorate(FunctionArguments ::= "(" ")") -> Vector<Sequence<SemanticExpression>, Sequence<SemanticProperty>>
+	:= [
+		[],
+		[],
+	];
+Decorate(FunctionArguments ::= "(" ","? Expression<+Block><?Break><?Return># ","? ")") -> Vector<Sequence<SemanticExpression>, Sequence<SemanticProperty>>
+	:= [
+		ParseList(Expression<+Block><?Break><?Return>, SemanticExpression),
+		[],
+	];
+Decorate(FunctionArguments ::= "(" ","? Property<?Break><?Return># ","? ")") -> Vector<Sequence<SemanticExpression>, Sequence<SemanticProperty>>
+	:= [
+		[],
+		ParseList(Property<?Break><?Return>, SemanticProperty),
+	];
+Decorate(FunctionArguments ::= "(" ","? Expression<+Block><?Break><?Return># "," Property<?Break><?Return># ","? ")") -> Vector<Sequence<SemanticExpression>, Sequence<SemanticProperty>>
+	:= [
+		ParseList(Expression<+Block><?Break><?Return>, SemanticExpression),
+		ParseList(Property<?Break><?Return>,           SemanticProperty),
+	];



Decorate(ExpressionCompound<Block, Break, Return> ::= ExpressionCompound<?Block><?Break><?Return> "." FunctionArguments<?Break><?Return>) -> SemanticCall
	:= (SemanticCall
		Decorate(ExpressionCompound<?Block><?Break><?Return>)
-		...Decorate(FunctionArguments<?Break><?Return>)
+		...Decorate(FunctionArguments<?Break><?Return>).0
+		...Decorate(FunctionArguments<?Break><?Return>).1
	);
Decorate(ExpressionCompound<Block, Break, Return> ::= ExpressionCompound<?Block><?Break><?Return> "." GenericArguments FunctionArguments<?Break><?Return>) -> SemanticCall
	:= (SemanticCall
		Decorate(ExpressionCompound<?Block><?Break><?Return>)
		...Decorate(GenericArguments)
-		...Decorate(FunctionArguments<?Break><?Return>)
+		...Decorate(FunctionArguments<?Break><?Return>).0
+		...Decorate(FunctionArguments<?Break><?Return>).1
	);
```
