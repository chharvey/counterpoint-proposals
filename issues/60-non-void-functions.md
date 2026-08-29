Non-void functions return values.

# Discussion
This issue adds the ability for functions to return values to their caller. It adds return types, nonempty `return` statements, and implicit returns.
```cpl
func add(a: int, b: int): int {
%                         ^ return type
	return a + b;
	% ^ nonempty return statement (with an expression)
}
func subtract(a: int, b: int): int => a - b;
%                                  ^ implicit return
```

When a function returns, it completes execution and returns control back to the caller where the call occurs. #46 covered **void functions**, which return but do not return a value. Their return “type” is `void` (though `void` is not actually a type). When a function returns *a value*, it sends a value along with control back to the caller. The **return type** of a function is the static type of the returned value(s), and it tells the compiler the type of the call expression (v0.7.0). If a function returns a value, then its body, the statement block, must have a **nonempty `return` statement** in each control flow branch, which contains the expression to evaluate and return. A function may have no body but an **implicit return** (using a fat arrow `=>`), which is the single expression that is returned. A function with an implicit return cannot contain any statements.

The above applies to function expressions as well.
```cpl
val math: (
	add:      \(a: int, b: int) => int,
	subtract: \(x: int, y: int) => int,
	multiply: \(int, int)       => int,
	divide:   \(int, int)       => int,
	%                              ^ return type
) = (
	add= \($a: int, $b: int): int {
		"""The sum will be {{ a + b }}.""";
		return a + b;
	},
	subtract= \(x= a: int, y= b: int): int => a - b,
	multiply= \(x: int, y: int): int => x * y;
	divide=   \(x: int, y: int): int {
		if y === 0 then {
			return 7;
		} else {
			return x / y;
		};
	},
);
```

## Variance
Function return types are **covariant**. This means that when assigning a function `g` to a function type `F`, the return type of `g` must be assignable to the return type of `F`.
```cpl
type BinaryOperator = \(int | float, int | float) => int | float;
val subtract: BinaryOperator = \(minuend: int | float, subtrahend: int | float): float => minuend - subtrahend;
```
When a caller calls an implementation of `BinaryOperator`, they should expect its return value to be assignable to `int | float`. Since the return type of `subtract` is narrower, it satisfies that requirement.

# Specification

## Syntax
```diff
TypeFunction
-	::= "\" "(" ParametersType? ")" "=>"  "void";
+	::= "\" "(" ParametersType? ")" "=>" ("void" | Type);

ExpressionFunction
-	::= "\" "(" ParametersFunction? ")" ":"  "void"          Block<-Break><+Return>;
+	::= "\" "(" ParametersFunction? ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression<+Block><-Break><+Return>);

-StatementReturn        ::= "return"                                      ";";
+StatementReturn<Break> ::= "return" Expression<+Block><?Break><+Return>? ";";

Statement<Break, Return> ::=
	| StatementExpression<?Break><?Return>
	| StatementConditional<∓Unless><?Break><?Return>
	| StatementLoop<Return>
	| StatementIteration<Return>
	| <Break+> StatementBreak
-	| <Return+>StatementReturn
+	| <Return+>StatementReturn<?Break>
	| Declaration
;

DeclarationFunction
-	::= "func" ("_" | IDENTIFIER) "(" ParametersFunction? ")" ":"  "void"          Block<-Break><+Return>
+	::= "func" ("_" | IDENTIFIER) "(" ParametersFunction? ")" ":" ("void" | Type) (Block<-Break><+Return> | "=>" Expression<+Block><-Break><+Return> ";")
```

## Semantics
```diff
SemanticTypeFunction
-	::= SemanticItemType* SemanticPropertyType*;
+	::= SemanticItemType* SemanticPropertyType* SemanticType?;

SemanticExpressionFunction
-	::= SemanticParameter*                        SemanticBlock;
+	::= SemanticParameterFunction* SemanticType? (SemanticBlock | SemanticExpression);

SemanticDeclarationFunction[id?: RealNumber]
-	::= SemanticParameterFunction*                SemanticBlock;
+	::= SemanticParameterFunction* SemanticType? (SemanticBlock | SemanticExpression);

SemanticStatementReturn
-	::= ();
+	::= SemanticExpression?;
```

## Decorate
```diff
Decorate(TypeFunction ::= "\" "(" ")" "=>" "void") -> SemanticTypeFunction
	:= (SemanticTypeFunction);
+Decorate(TypeFunction ::= "\" "(" ")" "=>" Type) -> SemanticTypeFunction
+	:= (SemanticTypeFunction Decorate(Type));
Decorate(TypeFunction ::= "\" "(" ParametersType ")" "=>" "void") -> SemanticTypeFunction
	:= (SemanticTypeFunction ...Decorate(ParametersType));
+Decorate(TypeFunction ::= "\" "(" ParametersType ")" "=>" Type) -> SemanticTypeFunction
+	:= (SemanticTypeFunction
+		...Decorate(ParametersType)
+		Decorate(Type)
+	);

Decorate(ExpressionFunction ::= "\" "(" ")" ":" "void" Block<-Break><+Return>) -> SemanticExpressionFunction
	:= (SemanticExpressionFunction Decorate(Block<-Break><+Return>));
+Decorate(ExpressionFunction ::= "\" "(" ")" ":" "void" "=>" Expression<+Block><-Break><+Return>) -> SemanticExpressionFunction
+	:= (SemanticExpressionFunction Decorate(Expression<+Block><-Break><+Return>));
+Decorate(ExpressionFunction ::= "\" "(" ")" ":" Type Block<-Break><+Return>) -> SemanticExpressionFunction
+	:= (SemanticExpressionFunction
+		Decorate(Type)
+		Decorate(Block<-Break><+Return>)
+	);
+Decorate(ExpressionFunction ::= "\" "(" ")" ":" Type "=>" Expression<+Block><-Break><+Return>) -> SemanticExpressionFunction
+	:= (SemanticExpressionFunction
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
Decorate(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" "void" Block<-Break><+Return>) -> SemanticExpressionFunction
	:= (SemanticExpressionFunction
		...Decorate(ParametersFunction)
		Decorate(Block<-Break><+Return>)
	);
+Decorate(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" "void" "=>" Expression<+Block><-Break><+Return>) -> SemanticExpressionFunction
+	:= (SemanticExpressionFunction
+		...Decorate(ParametersFunction)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
+Decorate(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" Type Block<-Break><+Return>) -> SemanticExpressionFunction
+	:= (SemanticExpressionFunction
+		...Decorate(ParametersFunction)
+		Decorate(Type)
+		Decorate(Block<-Break><+Return>)
+	);
+Decorate(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" Type "=>" Expression<+Block><-Break><+Return>) -> SemanticExpressionFunction
+	:= (SemanticExpressionFunction
+		...Decorate(ParametersFunction)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);

-Decorate(StatementReturn        ::= "return" ";") -> SemanticStatementReturn
+Decorate(StatementReturn<Break> ::= "return" ";") -> SemanticStatementReturn
	:= (SemanticStatementReturn);
+Decorate(StatementReturn<Break> ::= "return" Expression<+Block><?Break><+Return> ";") -> SemanticStatementReturn
+	:= (SemanticStatementReturn Decorate(Expression<+Block><?Break><+Return>));

-Decorate(Statement<Break, Return> ::= <Return+>StatementReturn)         -> SemanticStatementReturn
+Decorate(Statement<Break, Return> ::= <Return+>StatementReturn<?Break>) -> SemanticStatementReturn
-	:= Decorate(StatementReturn);
+	:= Decorate(StatementReturn<?Break>);

Decorate(DeclarationFunction ::= "func" "_" "(" ")" ":" "void" Block<-Break><+Return>) -> SemanticDeclarationFunction
	:= (SemanticDeclarationFunction[id=*nil*]
		Decorate(Block<-Break><+Return>)
	);
+Decorate(DeclarationFunction ::= "func" "_" "(" ")" ":" "void" "=>" Expression<+Block><-Break><+Return> ";") -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=*nil*]
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
+Decorate(DeclarationFunction ::= "func" "_" "(" ")" ":" Type Block<-Break><+Return>) -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=*nil*]
+		Decorate(Type)
+		Decorate(Block<-Break><+Return>)
+	);
+Decorate(DeclarationFunction ::= "func" "_" "(" ")" ":" Type "=>" Expression<+Block><-Break><+Return> ";") -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=*nil*]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
Decorate(DeclarationFunction ::= "func" "_" "(" ParametersFunction ")" ":" "void" Block<-Break><+Return>) -> SemanticDeclarationFunction
	:= (SemanticDeclarationFunction[id=*nil*]
		...Decorate(ParametersFunction)
		Decorate(Block<-Break><+Return>)
	);
+Decorate(DeclarationFunction ::= "func" "_" "(" ParametersFunction ")" ":" "void" "=>" Expression<+Block><-Break><+Return> ";") -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=*nil*]
+		...Decorate(ParametersFunction)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
+Decorate(DeclarationFunction ::= "func" "_" "(" ParametersFunction ")" ":" Type Block<-Break><+Return>) -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=*nil*]
+		...Decorate(ParametersFunction)
+		Decorate(Type)
+		Decorate(Block<-Break><+Return>)
+	);
+Decorate(DeclarationFunction ::= "func" "_" "(" ParametersFunction ")" ":" Type "=>" Expression<+Block><-Break><+Return> ";") -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=*nil*]
+		...Decorate(ParametersFunction)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
Decorate(DeclarationFunction ::= "func" IDENTIFIER "(" ")" ":" "void" Block<-Break><+Return>) -> SemanticDeclarationFunction
	:= (SemanticDeclarationFunction[id=TokenWorth(IDENTIFIER)]
		Decorate(Block<-Break><+Return>)
	);
+Decorate(DeclarationFunction ::= "func" IDENTIFIER "(" ")" ":" "void" "=>" Expression<+Block><-Break><+Return> ";") -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=TokenWorth(IDENTIFIER)]
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
+Decorate(DeclarationFunction ::= "func" IDENTIFIER "(" ")" ":" Type Block<-Break><+Return>) -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=TokenWorth(IDENTIFIER)]
+		Decorate(Type)
+		Decorate(Block<-Break><+Return>)
+	);
+Decorate(DeclarationFunction ::= "func" IDENTIFIER "(" ")" ":" Type "=>" Expression<+Block><-Break><+Return> ";") -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=TokenWorth(IDENTIFIER)]
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
Decorate(DeclarationFunction ::= "func" IDENTIFIER "(" ParametersFunction ")" ":" "void" Block<-Break><+Return>) -> SemanticDeclarationFunction
	:= (SemanticDeclarationFunction[id=TokenWorth(IDENTIFIER)]
		...Decorate(ParametersFunction)
		Decorate(Block<-Break><+Return>)
	);
+Decorate(DeclarationFunction ::= "func" IDENTIFIER "(" ParametersFunction ")" ":" "void" "=>" Expression<+Block><-Break><+Return> ";") -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=TokenWorth(IDENTIFIER)]
+		...Decorate(ParametersFunction)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
+Decorate(DeclarationFunction ::= "func" IDENTIFIER "(" ParametersFunction ")" ":" Type Block<-Break><+Return>) -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=TokenWorth(IDENTIFIER)]
+		...Decorate(ParametersFunction)
+		Decorate(Type)
+		Decorate(Block<-Break><+Return>)
+	);
+Decorate(DeclarationFunction ::= "func" IDENTIFIER "(" ParametersFunction ")" ":" Type "=>" Expression<+Block><-Break><+Return> ";") -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction[id=TokenWorth(IDENTIFIER)]
+		...Decorate(ParametersFunction)
+		Decorate(Type)
+		Decorate(Expression<+Block><-Break><+Return>)
+	);
```
