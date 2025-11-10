Non-void functions return values.

# Discussion
This issue adds the ability for functions to return values to their caller. It adds return types, `return` statements, and implicit returns.
```point
function add(a: int, b: int): int {
%                             ^ return type
	return a + b;
	% ^ return statement
}
function subtract(a: int, b: int): int => a - b;
%                                      ^ implicit return
```

When a function returns, it completes execution and returns control back to the caller where the call occurs. When a function returns *a value*, it sends a value along with control back to the caller. The **return type** of a function is the static type of the returned value, and it tells the compiler the type of the call expression (v0.7.0). If a function returns a value, then its body, the statement block, must have a **`return` statement**, which contains the expression to evaluate and return. A function may have no body but an **implicit return** (using a fat arrow `=>`), which is the single expression that is returned. A function with an implicit return cannot contain any statements.

The above applies to function expressions as well.
```point
let math: [
	add:      \(a: int, b: int) => int,
	subtract: \(x: int, y: int) => int,
	multiply: \(int, int)       => int,
	%                              ^ return type
] = [
	add= \(a: int, b: int): int {
		"""The sum will be {{ a + b }}.""";
		return a + b;
	},
	subtract= \(x as a: int, y as b: int): int => a - b,
	multiply= \(x: int, y: int): int => x * y;
];
```

A function body may have no return statement, or it may have an empty return statement (`return;` with no expression), in which case the function “returns void” (it returns, but it does not return a value) and its return type is `void`. Functions with implicit returns cannot return void — they must return a value. Functions that do not return (do not complete execution) are not covered in this issue, nor are asynchronous functions.
```point
function divide(a: int, b: int): void {
	"""I am not going to divide {{ a }} and {{ b }}.""";
	% automatically returns here since it’s the end of the function body
}
let exp: Object = (a: int, b: int): void {
	"""{{ a }} to the {{ b }} power is {{ a ^ b }}.""";
	return; % explicit empty return statement
};
```

## Variance
Function return types are **covariant**. This means that when assigning a function `g` to a function type `F`, the return type of `g` must be assignable to the return type of `F`.
```point
type BinaryOperator = \(int | float, int | float) => int | float;
let subtract: BinaryOperator = \(minuend: int | float, subtrahend: int | float): float => minuend - subtrahend;
```
When a caller calls an implementation of `BinaryOperator`, they should expect its return value to be assignable to `int | float`. Since the return type of `subtract` is narrower, it satisfies that requirement.

# Specification

## Syntax
```diff
TypeFunction<Named>
-	::= "\" "(" ","? ParameterType<?Named># ","? ")" "=>"  "void";
+	::= "\" "(" ","? ParameterType<?Named># ","? ")" "=>" ("void" | Type);

Statement<Break, Return> ::=
	| Expression? ";"
	| StatementAssignment
	| StatementIf    <?Break>
	| StatementUnless<?Break>
	| StatementWhile
	| StatementUntil
	| StatementFor
	| <Break+>StatementBreak
	| <Break+>StatementContinue
-	| <Return+>("return"             ";");
+	| <Return+>("return" Expression? ";");
	| Declaration
;

-DeclaredFunction   ::=     "(" ","? ParameterFunction# ","? ")" ":"  "void"          Block<-Break><+Return>;
-ExpressionFunction ::= "\" "(" ","? ParameterFunction# ","? ")" ":"  "void"          Block<-Break><+Return>;
+DeclaredFunction   ::=     "(" ","? ParameterFunction# ","? ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression ";");
+ExpressionFunction ::= "\" "(" ","? ParameterFunction# ","? ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression);

DeclarationFunction
-	::= "func" IDENTIFIER ExpressionFunction;
+	::= "func" IDENTIFIER DeclaredFunction;
```

## Semantics
```diff
SemanticTypeFunction
-	::= SemanticParameterType*;
+	::= SemanticParameterType* SemanticType;

SemanticFunction
-	::= SemanticParameter*               SemanticBlock;
+	::= SemanticParameter* SemanticType (SemanticBlock | SemanticExpression);

+SemanticReturn
+	::= SemanticExpression?;
```

## Decorate
```diff
+Decorate(TypeFunction<Named> ::= "\" "(" ","? ParameterType<?Named># ","? ")" "=>" Type) -> SemanticTypeFunction
+	:= (SemanticTypeFunction
+		...ParseList(ParameterType<?Named>, SemanticParameterType)
+		Decorate(Type)
+	);

+Decorate(DeclaredFunction ::= "(" ","? ParameterFunction# ","? ")" ":" Type Block<-Break>) -> SemanticFunction
+	:= (SemanticFunction
+		...ParseList(ParameterFunction, SemanticParameter)
+		Decorate(Type)
+		Decorate(Block<-Break>)
+	);
+Decorate(DeclaredFunction ::= "(" ","? ParameterFunction# ","? ")" ":" Type "=>" Expression ";") -> SemanticFunction
+	:= (SemanticFunction
+		...ParseList(ParameterFunction, SemanticParameter)
+		Decorate(Type)
+		Decorate(Expression)
+	);

+Decorate(ExpressionFunction ::= "\" "(" ","? ParameterFunction# ","? ")" ":" Type Block<-Break>) -> SemanticFunction
+	:= (SemanticFunction
+		...ParseList(ParameterFunction, SemanticParameter)
+		Decorate(Type)
+		Decorate(Block<-Break>)
+	);
+Decorate(ExpressionFunction ::= "\" "(" ","? ParameterFunction# ","? ")" ":" Type "=>" Expression) -> SemanticFunction
+	:= (SemanticFunction
+		...ParseList(ParameterFunction, SemanticParameter)
+		Decorate(Type)
+		Decorate(Expression)
+	);

+Decorate(Statement<+Return> ::= <Return+>("return" ";")) -> SemanticReturn
+	:= (SemanticReturn);
+Decorate(Statement<+Return> ::= <Return+>("return" Expression ";")) -> SemanticReturn
+	:= (SemanticReturn Decorate(Expression));

-Decorate(DeclarationFunction ::= "func" IDENTIFIER ExpressionFunction) -> SemanticDeclarationFunction
+Decorate(DeclarationFunction ::= "func" IDENTIFIER DeclaredFunction)   -> SemanticDeclarationFunction
	:= (SemanticDeclarationFunction
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
-		FunctionTypeOf(ExpressionFunction)
-		Decorate(ExpressionFunction)
+		FunctionTypeOf(DeclaredFunction)
+		Decorate(DeclaredFunction)
	);

-Decorate(Block<Break>         ::= "{" Statement<?Break>+          "}") -> SemanticBlock
+Decorate(Block<Break, Return> ::= "{" Statement<?Break><?Return>+ "}") -> SemanticBlock
	:= (SemanticBlock
-		...ParseList(Statement<?Break>,          SemanticStatement)
+		...ParseList(Statement<?Break><?Return>, SemanticStatement)
	);
```

## FunctionTypeOf
```diff
+FunctionTypeOf(DeclaredFunction ::= "(" ","? ParameterFunction# ","? ")" ":" Type Block<-Break>) -> SemanticTypeFunction
+	:= (SemanticTypeFunction
+		...FunctionTypeOf(ParameterFunction#)
+		Decorate(Type)
+	);
+FunctionTypeOf(DeclaredFunction ::= "(" ","? ParameterFunction# ","? ")" ":" Type "=>" Expression ";") -> SemanticTypeFunction
+	:= (SemanticTypeFunction
+		...FunctionTypeOf(ParameterFunction#)
+		Decorate(Type)
+	);

+FunctionTypeOf(ExpressionFunction ::= "\" "(" ","? ParameterFunction# ")" ":" Type Block<-Break>) -> SemanticTypeFunction
+	:= (SemanticTypeFunction
+		...FunctionTypeOf(ParameterFunction#)
+		Decorate(Type)
+	);
+FunctionTypeOf(ExpressionFunction ::= "\" "(" ","? ParameterFunction# ")" ":" Type "=>" Expression) -> SemanticTypeFunction
+	:= (SemanticTypeFunction
+		...FunctionTypeOf(ParameterFunction#)
+		Decorate(Type)
+	);
```
