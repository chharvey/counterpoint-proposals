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
TypeFunction
-	::= "\" "(" ParametersType? ")" "=>"  "void";
+	::= "\" "(" ParametersType? ")" "=>" ("void" | Type);

-StatementReturn        ::= "return"                                      ";";
+StatementReturn<Break> ::= "return" Expression<+Block><?Break><+Return>? ";";

Statement<Break, Return> ::=
	| StatementExpression<?Break><?Return>
	| StatementConditional<∓Unless><?Break><?Return>
	| StatementLoop<Return>
	| StatementIteration<Return>
	| <Break+> StatementBreak
	| <Break+> StatementContinue
-	| <Return+>StatementReturn
+	| <Return+>StatementReturn<?Break>
	| Declaration
;

-DeclaredFunction   ::=     "(" ParametersFunction? ")" ":"  "void"          Block<-Break><+Return>;
-ExpressionFunction ::= "\" "(" ParametersFunction? ")" ":"  "void"          Block<-Break><+Return>;
+DeclaredFunction   ::=     "(" ParametersFunction? ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression<+Block><-Break><+Return> ";");
+ExpressionFunction ::= "\" "(" ParametersFunction? ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression<+Block><-Break><+Return>);

DeclarationFunction
-	::= "func" IDENTIFIER ExpressionFunction;
+	::= "func" IDENTIFIER DeclaredFunction;
```

## Semantics
```diff
SemanticTypeFunction
-	::= SemanticItemType* SemanticPropertyType*;
+	::= SemanticItemType* SemanticPropertyType* SemanticType?;

SemanticFunction
-	::= SemanticParameter*                SemanticBlock;
+	::= SemanticParameter* SemanticType? (SemanticBlock | SemanticExpression);

SemanticReturn
-	::= ();
+	::= SemanticExpression?;
```

## Decorate
```diff
Decorate(TypeFunction ::= "\" "(" ")" "=>" "void") -> SemanticTypeFunction
	:= (SemanticTypeFunction);
Decorate(TypeFunction ::= "\" "(" ParametersType ")" "=>" "void") -> SemanticTypeFunction
	:= (SemanticTypeFunction ...Decorate(ParametersType));
+Decorate(TypeFunction ::= "\" "(" ")" "=>" Type) -> SemanticTypeFunction
+	:= (SemanticTypeFunction Decorate(Type));
+Decorate(TypeFunction ::= "\" "(" ParametersType ")" "=>" Type) -> SemanticTypeFunction
+	:= (SemanticTypeFunction
+		...Decorate(ParametersType)
+		Decorate(Type)
+	);

-Decorate(StatementReturn        ::= "return" ";") -> SemanticReturn
+Decorate(StatementReturn<Break> ::= "return" ";") -> SemanticReturn
	:= (SemanticReturn);
+Decorate(StatementReturn<Break> ::= "return" Expression<+Block><?Break><+Return> ";") -> SemanticReturn
+	:= (SemanticReturn Decorate(Expression<+Block><?Break><+Return>));

-Decorate(Statement<Break, Return> ::= <Return+>StatementReturn)         -> SemanticReturn
+Decorate(Statement<Break, Return> ::= <Return+>StatementReturn<?Break>) -> SemanticReturn
-	:= Decorate(StatementReturn);
+	:= Decorate(StatementReturn<?Break>);

Decorate(DeclaredFunction ::= "(" ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
	:= (SemanticFunction Decorate(Block<-Break><+Return>));
Decorate(DeclaredFunction ::= "(" ParametersFunction ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
	:= (SemanticFunction
		...Decorate(ParametersFunction)
		Decorate(Block<-Break><+Return>)
	);
+Decorate(DeclaredFunction ::= "(" ")" ":" "void" "=>" Expression<+Block><-Break><+Return> ";") -> SemanticFunction
+	:= (SemanticFunction Decorate(Expression<+Block><-Break><+Return>));
+Decorate(DeclaredFunction ::= "(" ParametersFunction ")" ":" "void" "=>" Expression<+Block><-Break><+Return> ";") -> SemanticFunction
+	:= (SemanticFunction
+		...Decorate(ParametersFunction)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
+Decorate(DeclaredFunction ::= "(" ")" ":" Type Block<-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction
+		Decorate(Type)
+		Decorate(Block<-Break><+Return>)
+	);
+Decorate(DeclaredFunction ::= "(" ParametersFunction ")" ":" Type Block<-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction
+		...Decorate(ParametersFunction)
+		Decorate(Type)
+		Decorate(Block<-Break><+Return>)
+	);
+Decorate(DeclaredFunction ::= "(" ")" ":" Type "=>" Expression<+Block><-Break><+Return> ";") -> SemanticFunction
+	:= (SemanticFunction
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
+Decorate(DeclaredFunction ::= "(" ParametersFunction ")" ":" Type "=>" Expression<+Block><-Break><+Return> ";") -> SemanticFunction
+	:= (SemanticFunction
+		...Decorate(ParametersFunction)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);

Decorate(ExpressionFunction ::= "\" "(" ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
	:= (SemanticFunction Decorate(Block<-Break><+Return>));
Decorate(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
	:= (SemanticFunction
		...Decorate(ParametersFunction)
		Decorate(Block<-Break><+Return>)
	);
+Decorate(ExpressionFunction ::= "\" "(" ")" ":" "void" "=>" Expression<+Block><-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction Decorate(Expression<+Block><-Break><+Return>));
+Decorate(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" "void" "=>" Expression<+Block><-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction
+		...Decorate(ParametersFunction)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
+Decorate(ExpressionFunction ::= "\" "(" ")" ":" Type Block<-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction
+		Decorate(Type)
+		Decorate(Block<-Break><+Return>)
+	);
+Decorate(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" Type Block<-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction
+		...Decorate(ParametersFunction)
+		Decorate(Type)
+		Decorate(Block<-Break><+Return>)
+	);
+Decorate(ExpressionFunction ::= "\" "(" ")" ":" Type "=>" Expression<+Block><-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
+Decorate(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" Type "=>" Expression<+Block><-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction
+		...Decorate(ParametersFunction)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);

-Decorate(Block<Break>         ::= "{" Statement<?Break>+          "}") -> SemanticBlock
+Decorate(Block<Break, Return> ::= "{" Statement<?Break><?Return>+ "}") -> SemanticBlock
	:= (SemanticBlock
-		...ParseList(Statement<?Break>,          SemanticStatement)
+		...ParseList(Statement<?Break><?Return>, SemanticStatement)
	);
```

## FunctionTypeOf
```diff
-FunctionTypeOf(DeclaredFunction ::= "(" ")" ":" "void"  Block<-Break>)                                                 -> SemanticTypeFunction
+FunctionTypeOf(DeclaredFunction ::= "(" ")" ":" "void" (Block<-Break> | "=>" Expression<+Block><-Break><+Return> ";")) -> SemanticTypeFunction
	:= (SemanticTypeFunction);
-FunctionTypeOf(DeclaredFunction ::= "(" ParametersFunction ")" ":" "void"  Block<-Break>)                                                 -> SemanticTypeFunction
+FunctionTypeOf(DeclaredFunction ::= "(" ParametersFunction ")" ":" "void" (Block<-Break> | "=>" Expression<+Block><-Break><+Return> ";")) -> SemanticTypeFunction
	:= (SemanticTypeFunction ...FunctionTypeOf(ParametersFunction));
+FunctionTypeOf(DeclaredFunction ::= "(" ")" ":" Type (Block<-Break> | "=>" Expression<+Block><-Break><+Return> ";")) -> SemanticTypeFunction
+	:= (SemanticTypeFunction Decorate(Type));
+FunctionTypeOf(DeclaredFunction ::= "(" ParametersFunction ")" ":" Type (Block<-Break> | "=>" Expression<+Block><-Break><+Return> ";")) -> SemanticTypeFunction
+	:= (SemanticTypeFunction
+		...FunctionTypeOf(ParametersFunction#)
+		Decorate(Type)
+	);

-FunctionTypeOf(ExpressionFunction ::= "\" "(" ")" ":" "void"  Block<-Break>) -> SemanticTypeFunction
+FunctionTypeOf(ExpressionFunction ::= "\" "(" ")" ":" "void" (Block<-Break> | "=>" Expression<+Block><-Break><+Return> ";")) -> SemanticTypeFunction
	:= (SemanticTypeFunction);
-FunctionTypeOf(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" "void"  Block<-Break>) -> SemanticTypeFunction
+FunctionTypeOf(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" "void" (Block<-Break> | "=>" Expression<+Block><-Break><+Return> ";")) -> SemanticTypeFunction
	:= (SemanticTypeFunction ...FunctionTypeOf(ParametersFunction));
+FunctionTypeOf(ExpressionFunction ::= "\" "(" ")" ":" Type (Block<-Break> | "=>" Expression<+Block><-Break><+Return> ";")) -> SemanticTypeFunction
+	:= (SemanticTypeFunction Decorate(Type));
+FunctionTypeOf(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" Type (Block<-Break> | "=>" Expression<+Block><-Break><+Return> ";")) -> SemanticTypeFunction
+	:= (SemanticTypeFunction
+		...FunctionTypeOf(ParametersFunction)
+		Decorate(Type)
+	);
```
