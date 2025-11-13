Define synchronous function declarations and function expressions.

# Discussion

## Function Definitions
This issue covers defining void synchronous functions only, which do return but do not return a value (#70). This issue does not cover non-void functions, asynchronous functions, or error throwing.

This is a function declaration:
```point
function add(a: int, b: int): void {
	"""
		The first argument is {{ a }}.
		The second argument is {{ b }}.
	""";
	"This function does not return a value.";
	return;
}
```
The **name** of this function is `add`. It has two input **parameters** `a` and `b`, each of type `int`. When a function is called (see v0.7.0), the values sent into the function are **arguments**. This function’s **return type** is `void`, because it has no output value. Its **body** is the set of statements within the curly braces. The body of a void function *must* include a `return;` statement in every code path.

This is a function expression (lambda):
```point
\(a: int, b: int): void {
	"""
		The first argument is {{ a }}.
		The second argument is {{ b }}.
	""";
	"This function does not return a value.";
	return;
};
```
Lambdas are normal expressions that can be operated on and passed around like any other value. For instance, lambdas can be assigned to variables. Lambdas are always “truthy”.
```point
let my_fn: \(a: int, b: int) => void =
	\(a: int, b: int): void { a + b; return; };
!!my_fn; %== true
```

Some parameters may be declared with `var`, which means they can be reassigned within the function body.
```point
function add(var a: int, b: int): void {
	set a = a + 1; % ok
	set b = b - 1; %> AssignmentError
	return;
}
```

Some parameters may be **named**, which means when the function is called (see v0.7.0) its arguments must have a corresponding name. The **exernal name** is before the equals sign and is what the caller will use to name their arguments. The **internal name** comes after the equal sign and is used as the parameter name in the function body.
```cpl
function add(alpha= a: int, bravo= var b: int): void {
	set b = b - 1;
	"""
		The first argument is {{ a }}.
		The second argument is {{ b }}.
	""";
	return;
}
```
When called, the caller must supply arguments for `alpha` and `bravo`.

If we want the external and internal names to be the same, we can use `$`-punning: `$a` is shorthand for `a= a`.
```cpl
function add($a: int, var $b: int): void {
	set b = b - 1;
	"""
		The first argument is {{ a }}.
		The second argument is {{ b }}.
	""";
	return;
}
```
Now the caller must supply arguments for `a` and `b`, which are the same names used in the function body.

A function can have named and unnamed (“positional”) parameters, but all named parameters *must* come after all positional parameters. It is a syntax error to have them not in that order. However, within the named parameters, punned and un-punned parameters may be intermixed.
```cpl
function foo(a: int, b: int, $c: int, delta= d: int, $e: int): void { return; }
```

## Type Signatures
A function’s **type signature** is its type, written in the form of `\(‹params›) => ‹return›`. It indicates what types of arguments are accepted and what type the function returns. This issue only covers functions that return `void`. The type of a positional parameter is just its type, whereas the type of a named parameter is `‹name›: ‹type›`. The type signature provides a contract to the caller of what’s expected in the arguments list.
```cpl
let my_fn: \(int, b: int) => void =
	\(a: int, $b: int): void { a + b; return; };

% typeof my_fn: \(int, b: int) => void
```

## Function Assignment
When assigning a function to a type signature with named parameters (in the case of type alias assignment or abstract method implementation), the assigned parameter order must match up with the assignee parameters.
```cpl
type BinaryOperatorType = \(first: int, second: float) => void;
let add: BinaryOperatorType = \($second: float, $first: int): void { first + second; return; }; %> TypeError
```
> TypeError: Type `\(second: float, first: int) => void` is not assignable to type `\(first: int, second: float) => void`.

The reason for this error is that one should expect to be able to call any `BinaryOperatorType` with the positional arguments of an `int` followed by a `float`. Calling it with e.g. `4.0` and `2`, in that order, should fail. From this perspective, function assignment is a bit like tuple assignment.

From another perspective, function assignment is like record assignment: the parameter names of the assigned must match the parameter names of the assignee.
```cpl
type BinaryOperatorType = \(first: float, second: float) => void;
let subtract: BinaryOperatorType = \($x: float, $y: float): void { x - y; return; }; %> TypeError
```
> TypeError: Type `\(x: float, y: float) => void` is not assignable to type `\(first: float, second: float) => void`.

This errors because a caller must be able to call `subtract` with the named arguments `first` and `second`.

Luckily, because parameters’ external names can be different from their internal names, this is a convenient way of handling function assignment/implementation mismatches.
```cpl
let subtract: BinaryOperatorType = \(first= x: float, second= y: float): void {
	first;  %> ReferenceError
	second; %> ReferenceError
	x - y;
	return;
};
```
This lets the function author internally use the parameter names `x` and `y` while still allowing the caller to call the function with named arguments `first` and `second` repectively, and now the assignment to type `BinaryOperatorType` is valid.


## Variance
Function parameter types are **contravariant**. This means that when assigning a function `g` to a function type `F`, the type of each parameter of `F` must be assignable to the corresponding parameter’s type of `g`.
```point
type UnaryOperator = \(float | str) => void;
let g: UnaryOperator = \(x: float): void { %> TypeError
	x; %: float
	return;
};
```
A type error is raised because we cannot assign a `\(float) => void` type to a `\(float | str) => void` type. Even though the parameter’s type is narrower, a caller should expect to be able to call any `UnaryOperator` implementation with a `str` argument, and our implementation doesn’t allow that.

However, we can *widen* the parameter types:
```point
let h: UnaryOperator = \(x: int | float | str): void {
	x; %: int | float | str
	return;
};
```
This meets the requirements of `UnaryOperator` but still has a wider type for its parameter.

# Specification

## Lexicon
```diff
Punctuator :::=
	// storage
+		| "\"
+		| "=>"
;

Keyword :::=
	// storage
+		| "function"
	// statement
+		| "return"
;
```

## Syntax
```diff
EntryType<Named, Optional>
	::= <Named+>(Word & <Optional->":") <Optional+>"?:" Type;

+TypeFunction
+	::= "\" "(" ParametersType? ")" "=>" "void";

Type ::=
	| TypeUnion
+	| TypeFunction
;

#... StringTemplate thru MapLiteral

-ExpressionUnit<Block, Break> ::=
+ExpressionUnit<Block, Break, Return> ::=
	| IDENTIFIER
	| PrimitiveLiteral
-	| StringTemplate    <?Break>
-	| ExpressionGrouped <?Break>
-	| TupleLiteral      <?Break>
-	| RecordLiteral     <?Break>
-	| ListLiteral       <?Break>
-	| DictLiteral       <?Break>
-	| SetLiteral        <?Break>
-	| MapLiteral        <?Break>
-	| <Block+>Block     <?Break>
+	| StringTemplate    <?Break><?Return>
+	| ExpressionGrouped <?Break><?Return>
+	| TupleLiteral      <?Break><?Return>
+	| RecordLiteral     <?Break><?Return>
+	| ListLiteral       <?Break><?Return>
+	| DictLiteral       <?Break><?Return>
+	| SetLiteral        <?Break><?Return>
+	| MapLiteral        <?Break><?Return>
+	| <Block+>Block     <?Break><?Return>
;

#... ExpressionCompound thru ExpressionConditional

-Expression<Block, Break> ::=
-	| ExpressionDisjunctive<?Block><?Break>
-	| ExpressionConditional<?Break>
+Expression<Block, Break, Return> ::=
+	| ExpressionDisjunctive<?Block><?Break><?Return>
+	| ExpressionConditional<?Break><?Return>
+	| ExpressionFunction
;


-StatementExpression<Break>         ::= Expression<+Block><?Break>?          ";";
+StatementExpression<Break, Return> ::= Expression<+Block><?Break><?Return>? ";";

-StatementConditional<Unless, Break> ::=
+StatementConditional<Unless, Break, Return> ::=
-	& <Unless->"if" <Unless+>"unless" Expression<+Block><?Break>
-	& "then" Block<?Break>
+	& <Unless->"if" <Unless+>"unless" Expression<+Block><?Break><?Return>
+	& "then" Block<?Break><?Return>
	& <Unless->(
-		| ("else" Block<?Break>)? ";"
-		| "else" StatementConditional<?Unless><?Break>
+		| ("else" Block<?Break><?Return>)? ";"
+		| "else" StatementConditional<?Unless><?Break><?Return>
	)
	& <Unless+>";"
;

-StatementLoop
+StatementLoop<Return>
-	::= (("while" | "until") Expression<+Block><-Break>          && "do" Block<+Break>) ";";
+	::= (("while" | "until") Expression<+Block><-Break><?Return> && "do" Block<+Break><?Return>) ";";

-StatementIteration
+StatementIteration<Return>
-	::= "for" ("_" | IDENTIFIER) ":" Type "of" Expression<+Block><-Break>          "do" Block<+Break> ";";
+	::= "for" ("_" | IDENTIFIER) ":" Type "of" Expression<+Block><-Break><?Return> "do" Block<+Break><?Return> ";";

StatementBreak    ::= "break"    INTEGER? ";";
StatementContinue ::= "continue" INTEGER? ";";
+StatementReturn  ::= "return"            ";";

-Statement<Break> ::=
-	| StatementExpression<?Break>
-	| StatementConditional<∓Unless><?Break>
-	| StatementLoop
-	| StatementIteration
+Statement<Break, Return> ::=
+	| StatementExpression<?Break><?Return>
+	| StatementConditional<∓Unless><?Break><?Return>
+	| StatementLoop<Return>
+	| StatementIteration<Return>
	| <Break+> StatementBreak
	| <Break+> StatementContinue
+	| <Return+>StatementReturn
	| Declaration
;

+DeclaredFunction   ::=     "(" ParametersFunction? ")" ":" "void" Block<-Break><+Return>;
+ExpressionFunction ::= "\" "(" ParametersFunction? ")" ":" "void" Block<-Break><+Return>;

-Block<Break>         ::= "{" Statement<?Break>*          "}";
+Block<Break, Return> ::= "{" Statement<?Break><?Return>* "}";

+ParameterType<Named>
+	::= EntryType<?Named><-Optional>;

+ParameterFunction<Named>
+	::= <Named->"var"? <Named+>(Word "=" "var"? | "var"? "$") ("_" | IDENTIFIER) ":" Type;

+ParametersType ::=
+	| ","? ParameterType<-Named># ("," ParameterType<+Named>#)? ","?
+	| ","?                             ParameterType<+Named>#   ","?
+;

+ParametersFunction ::=
+	| ","? ParameterFunction<-Named># ("," ParameterFunction<+Named>#)? ","?
+	| ","?                                 ParameterFunction<+Named>#   ","?
+;

+DeclarationFunction
+	::= "function" IDENTIFIER DeclaredFunction;

Declaration ::=
	| DeclarationVariable
	| DeclarationType
+	| DeclarationFunction
;
```

## Semantics
```diff
SemanticItemType[optional: Boolean]
	::= SemanticType;

SemanticPropertyType[optional: Boolean]
	::= SemanticKey SemanticType;

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
+	::= SemanticItemType* SemanticPropertyType*;

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

SemanticStatement =:=
	| SemanticStatementExpression
	| SemanticStatementConditional
	| SemanticLoop
	| SemanticIteration
	| SemanticBreak
	| SemanticContinue
+	| SemanticReturn
	| SemanticDeclaration
;

+SemanticReturn
+	::= ();

+SemanticFunction
+	::= SemanticParameter* SemanticBlock;

+SemanticParameter[unfixed: Boolean]
+	::= SemanticKey? SemanticVariable? SemanticType;

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
+Decorate(TypeFunction ::= "\" "(" ")" "=>" "void") -> SemanticTypeFunction
+	:= (SemanticTypeFunction);
+Decorate(TypeFunction ::= "\" "(" ParametersType ")" "=>" "void") -> SemanticTypeFunction
+	:= (SemanticTypeFunction ...Decorate(ParametersType));

+Decorate(Type ::= TypeFunction) -> SemanticTypeFunction
+	:= Decorate(TypeFunction);

+Decorate(Expression ::= ExpressionFunction) -> SemanticFunction
+	:= Decorate(ExpressionFunction);

Decorate(StatementBreak ::= "break"         ";") -> SemanticBreak := (SemanticBreak[times=1]);
Decorate(StatementBreak ::= "break" INTEGER ";") -> SemanticBreak := (SemanticBreak[times=TokenWorth(INTEGER)]);

Decorate(StatementContinue ::= "continue"         ";") -> SemanticContinue := (SemanticContinue[times=1]);
Decorate(StatementContinue ::= "continue" INTEGER ";") -> SemanticContinue := (SemanticContinue[times=TokenWorth(INTEGER)]);

+Decorate(StatementReturn ::= "return" ";") -> SemanticReturn := (SemanticReturn);

-Decorate(Statement<Break>         ::= StatementExpression<?Break>) -> SemanticStatementExpression
+Decorate(Statement<Break, Return> ::= StatementExpression<?Break>) -> SemanticStatementExpression
	:= Decorate(StatementExpression<?Break>);
-Decorate(Statement<Break>         ::= StatementConditional<∓Unless><?Break>) -> SemanticConditional
+Decorate(Statement<Break, Return> ::= StatementConditional<∓Unless><?Break>) -> SemanticConditional
	:= Decorate(StatementConditional<∓Unless><?Break>);
-Decorate(Statement<Break>         ::= StatementLoop) -> SemanticLoop
+Decorate(Statement<Break, Return> ::= StatementLoop) -> SemanticLoop
	:= Decorate(StatementLoop);
-Decorate(Statement<Break>         ::= StatementIteration) -> SemanticIteration
+Decorate(Statement<Break, Return> ::= StatementIteration) -> SemanticIteration
	:= Decorate(StatementIteration);
-Decorate(Statement<Break>         ::= <Break+>StatementBreak) -> SemanticBreak
+Decorate(Statement<Break, Return> ::= <Break+>StatementBreak) -> SemanticBreak
	:= Decorate(StatementBreak);
-Decorate(Statement<Break>         ::= <Break+>StatementContinue) -> SemanticContinue
+Decorate(Statement<Break, Return> ::= <Break+>StatementContinue) -> SemanticContinue
	:= Decorate(StatementContinue);
+Decorate(Statement<Break, Return> ::= <Return+>StatementReturn) -> SemanticReturn
+	:= Decorate(StatementReturn);

+Decorate(DeclaredFunction ::= "(" ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction Decorate(Block<-Break><+Return>));
+Decorate(DeclaredFunction ::= "(" ParametersFunction ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction
+		...Decorate(ParametersFunction)
+		Decorate(Block<-Break><+Return>)
+	);

+Decorate(ExpressionFunction ::= "\" "(" ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction Decorate(Block<-Break><+Return>));
+Decorate(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" "void" Block<-Break><+Return>) -> SemanticFunction
+	:= (SemanticFunction
+		...Decorate(ParametersFunction)
+		Decorate(Block<-Break><+Return>)
+	);

+Decorate(ParameterType<-Named> ::= EntryType<-Named><-Optional>) -> SemanticItemType
+	:= Decorate(EntryType<-Named><-Optional>);
+Decorate(ParameterType<+Named> ::= EntryType<+Named><-Optional>) -> SemanticPropertyType
+	:= Decorate(EntryType<+Named><-Optional>);

+Decorate(ParameterFunction<-Named> ::= "_" ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		Decorate(Type)
+	);
+Decorate(ParameterFunction<-Named> ::= IDENTIFIER ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);
+Decorate(ParameterFunction<-Named> ::= "var" "_" ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		Decorate(Type)
+	);
+Decorate(ParameterFunction<-Named> ::= "var" IDENTIFIER ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);
+Decorate(ParameterFunction<+Named> ::= "$" "_" ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticKey[id=TokenWorth("_")])
+		Decorate(Type)
+	);
+Decorate(ParameterFunction<+Named> ::= "$" IDENTIFIER ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		(SemanticKey[id=TokenWorth(IDENTIFIER)])
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);
+Decorate(ParameterFunction<+Named> ::= "var" "$" "_" ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticKey[id=TokenWorth("_")])
+		Decorate(Type)
+	);
+Decorate(ParameterFunction <+Named>::= "var" "$" IDENTIFIER ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		(SemanticKey[id=TokenWorth(IDENTIFIER)])
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);
+Decorate(ParameterFunction<+Named> ::= Word "=" "_" ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		Decorate(Word)
+		Decorate(Type)
+	);
+Decorate(ParameterFunction<+Named> ::= Word "=" IDENTIFIER ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=false]
+		Decorate(Word)
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);
+Decorate(ParameterFunction<+Named> ::= Word "=" "var" "_" ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		Decorate(Word)
+		Decorate(Type)
+	);
+Decorate(ParameterFunction<+Named> ::= Word "=" "var" IDENTIFIER ":" Type) -> SemanticParameter
+	:= (SemanticParameter[unfixed=true]
+		Decorate(Word)
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		Decorate(Type)
+	);

+Decorate(ParametersType ::= ","? ParameterType<-Named># ","?) -> Sequence<SemanticItemType>
+	:= ParseList(ParameterType<-Named>, SemanticItemType)
+Decorate(ParametersType ::= ","? ParameterType<+Named># ","?) -> Sequence<SemanticPropertyType>
+	:= ParseList(ParameterType<+Named>, SemanticPropertyType)
+Decorate(ParametersType ::= ","? ParameterType<-Named># "," ParameterType<+Named># ","?) -> Sequence<...Sequence<SemanticItemType>, ...Sequence<SemanticPropertyType>>
+	:= [
+		...ParseList(ParameterType<-Named>, SemanticItemType),
+		...ParseList(ParameterType<+Named>, SemanticPropertyType),
+	];

+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named># ","?) -> Sequence<SemanticParameter>
+	:= ParseList(ParameterFunction<-Named>, SemanticParameter)
+Decorate(ParametersFunction ::= ","? ParameterFunction<+Named># ","?) -> Sequence<SemanticParameter>
+	:= ParseList(ParameterFunction<+Named>, SemanticParameter)
+Decorate(ParametersFunction ::= ","? ParameterFunction<-Named># "," ParameterFunction<+Named># ","?) -> Sequence<SemanticParameter>
+	:= [
+		...ParseList(ParameterFunction<-Named>, SemanticParameter),
+		...ParseList(ParameterFunction<+Named>, SemanticParameter),
+	];

+Decorate(DeclarationFunction ::= "func" IDENTIFIER DeclaredFunction) -> SemanticDeclarationFunction
+	:= (SemanticDeclarationFunction
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		FunctionTypeOf(DeclaredFunction)
+		Decorate(DeclaredFunction)
+	);

+Decorate(Declaration ::= DeclarationFunction) -> SemanticDeclarationFunction
+	:= Decorate(DeclarationFunction);
```

## FunctionTypeOf
```diff
+FunctionTypeOf(DeclaredFunction ::= "(" ")" ":" "void" Block<-Break>) -> SemanticTypeFunction
+	:= (SemanticTypeFunction);
+FunctionTypeOf(DeclaredFunction ::= "(" ParametersFunction ")" ":" "void" Block<-Break>) -> SemanticTypeFunction
+	:= (SemanticTypeFunction ...FunctionTypeOf(ParametersFunction));

+FunctionTypeOf(ExpressionFunction ::= "\" "(" ")" ":" "void" Block<-Break>) -> SemanticTypeFunction
+	:= (SemanticTypeFunction);
+FunctionTypeOf(ExpressionFunction ::= "\" "(" ParametersFunction ")" ":" "void" Block<-Break>) -> SemanticTypeFunction
+	:= (SemanticTypeFunction ...FunctionTypeOf(ParametersFunction));

+	FunctionTypeOf(ParametersFunction ::= ParameterFunction<-Named># ","?) -> Sequence<SemanticParameter>
+		:= FunctionTypeOf(ParameterFunction<-Named>#);
+	FunctionTypeOf(ParametersFunction ::= ParameterFunction<+Named># ","?) -> Sequence<SemanticParameter>
+		:= FunctionTypeOf(ParameterFunction<+Named>#);
+	FunctionTypeOf(ParametersFunction ::= ParameterFunction<-Named># "," ParameterFunction<+Named># ","?) -> Sequence<SemanticParameter>
+		:= [
+			...FunctionTypeOf(ParameterFunction<-Named>#),
+			...FunctionTypeOf(ParameterFunction<+Named>#),
+		];

+	FunctionTypeOf(ParameterFunction<Named># ::= ParameterFunction<?Named>) -> Tuple<SemanticItemType | SemanticPropertyType>
+		:= [FunctionTypeOf(ParameterFunction<?Named>)];
+	FunctionTypeOf(ParameterFunction<Named># ::= ParameterFunction<?Named># "," ParameterFunction<?Named>) -> Sequence<SemanticItemType | SemanticPropertyType>
+		:= [
+			...FunctionTypeOf(ParameterFunction<?Named>#),
+			FunctionTypeOf(ParameterFunction<?Named>),
+		];

+FunctionTypeOf(ParameterFunction<-Named> ::= "var"? ("_" | IDENTIFIER) ":" Type) -> SemanticItemType
+	:= (SemanticItemType[optionanl=false] Decorate(Type));
+FunctionTypeOf(ParameterFunction<+Named> ::= "var"? "$" ("_" | IDENTIFIER) ":" Type) -> SemanticPropertyType
+	:= (SemanticPropertyType[optional=false]
+		(SemanticKey[id=TokenWorth("_" | IDENTIFIER)])
+		Decorate(Type)
+	);
+FunctionTypeOf(ParameterFunction ::= Word "=" "var"? ("_" | IDENTIFIER) ":" Type) -> SemanticPropertyType
+	:= (SemanticPropertyType[optional=false]
+		Decorate(Word)
+		Decorate(Type)
+	);
```
