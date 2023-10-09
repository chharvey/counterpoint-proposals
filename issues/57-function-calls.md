Syntax for calling functions.

# Discussion
This issue covers only the syntax and compile-time semantics of calling synchronous functions. All functions in this version are synchronous. Asynchronous and generator functions are in the next version.

A function call has the syntax `‹f›.(‹args›)`, where `‹f›` is a compound expression representing a function, and `‹args›` are zero or more arguments passed into the function.

Function arguments may be a mix of **positional arguments** and **named arguments**, but all positional arguments *must* come before all named arguments in a function call. Positional arguments are simply expressions, and named arguments are expressions followed by a label, akin to a record property: `label= (argument)`.

All of the function calls below are equivalent to `subtract.(a= 5, b= 3)`.
```cp
function subtract(a: int, b: int): int {
	return a - b;
}
subtract.(5, 3);       %== 2
subtract.(5, b= 3);    %== 2
subtract.(a= 5, b= 3); %== 2
subtract.(b= 3, a= 5); %== 2
```

Arguments are assigned to parameters at compile-time. All positional arguments are assigned first, followed by all named arguments. Any doubly-asigned arguments override each other in source order. Any required parameters left unassigned will result in a type error, as well as any unassigned arguments.
```cp
subtract.(5);                %> TypeError: argument `b` required
subtract.(5, a= 3);          %> TypeError: argument `b` required
subtract.(b= 5, c= 3);       %> TypeError: argument `a` required
subtract.(a= 5, b= 3, c= 2); %> TypeError: too many arguments
subtract.(5, 3, 2);          %> TypeError: too many arguments

subtract.(2, b= 3, a= 5);    % same as `subtract.(a= 5, b= 3)~`
subtract.(b= 2, a= 5, b= 3); % same as `subtract.(a= 5, b= 3)~`
```

After all arguments are assigned, and assuming no type errors, any unassigned optional parameters are then evaluated. This happens in the function *call*, not in its definition.
```cp
function multiply(a: float, b: float = 1.0): float {
	return a * b;
};
multiply.(2.0);    % evaluates default `b` here
multiply.(a= 2.0); % evaluates default `b` here again
```

At runtime, arguments are always evaluated first, in left-to-right order, before they are sent into the function and the function is executed. The evaluation of optional parameters happens at the *call site*, but within the scope of the *definition site*. This means that any variables within an optional parameter’s default value are bound to the environment in which the function was defined. See #55 for details.

# Specification

## Syntax
```diff
Property ::= Word       "="  Expression;
Case     ::= Expression "->" Expression;

FunctionArguments ::=
	| "("                                        ")"
	| "(" ","?  Expression#                 ","? ")"
+	| "(" ","? (Expression# ",")? Property# ","? ")"
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
-Decorate(FunctionArguments ::= "(" ( ","? Expression# ","? )? ")") -> Sequence<SemanticExpression>
-	:= ParseList(Expression, SemanticExpression);
+Decorate(FunctionArguments ::= "(" ")") -> Vector<Sequence<SemanticExpression>, Sequence<SemanticProperty>>
+	:= [
+		[],
+		[],
+	];
+Decorate(FunctionArguments ::= "(" ","? Expression# ","? ")") -> Vector<Sequence<SemanticExpression>, Sequence<SemanticProperty>>
+	:= [
+		ParseList(Expression, SemanticExpression),
+		[],
+	];
+Decorate(FunctionArguments ::= "(" ","? Property# ","? ")") -> Vector<Sequence<SemanticExpression>, Sequence<SemanticProperty>>
+	:= [
+		[],
+		ParseList(Property, SemanticProperty),
+	];
+Decorate(FunctionArguments ::= "(" ","? Expression# "," Property# ","? ")") -> Vector<Sequence<SemanticExpression>, Sequence<SemanticProperty>>
+	:= [
+		ParseList(Expression, SemanticExpression),
+		ParseList(Property, SemanticProperty),
+	];

-Decorate(FunctionCall ::= "." FunctionArguments) -> Vector<Sequence<SemanticType>, Sequence<SemanticExpression>>
+Decorate(FunctionCall ::= "." FunctionArguments) -> Vector<Sequence<SemanticType>, Vector<Sequence<SemanticExpression>, Sequence<SemanticProperty>>>
	:= [
		[],
		Decorate(FunctionArguments),
	];
-Decorate(FunctionCall ::= "." GenericArguments FunctionArguments) -> Vector<Sequence<SemanticType>, Sequence<SemanticExpression>>
+Decorate(FunctionCall ::= "." GenericArguments FunctionArguments) -> Vector<Sequence<SemanticType>, Vector<Sequence<SemanticExpression>, Sequence<SemanticProperty>>>
	:= [
		Decorate(GenericArguments),
		Decorate(FunctionArguments),
	];

Decorate(ExpressionCompound ::= ExpressionUnit) -> SemanticExpression
	:= Decorate(ExpressionUnit);
Decorate(ExpressionCompound ::= ExpressionCompound PropertyAccess) -> SemanticAccess
	:= (SemanticAccess[kind=AccessKind(PropertyAccess)]
		Decorate(ExpressionCompound)
		Decorate(PropertyAccess)
	);
Decorate(ExpressionCompound ::= ExpressionCompound FunctionCall) -> SemanticCall
	:= (SemanticCall
		Decorate(ExpressionCompound)
-		...(...Decorate(FunctionCall))
+		...Decorate(FunctionCall).0
+		...Decorate(FunctionCall).1.0
+		...Decorate(FunctionCall).1.1
	);
```
