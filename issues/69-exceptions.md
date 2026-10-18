The Exception type is a new built-in type. It may be thrown by functions.

# Discussion

## Exception Type
In this version, the Exception type is a new struct type that has one field, `message`, which is a string. Once we get classes, `Exception` will be a class with its own constructor, but for now, “Exception objects” are constructed via a custom `Exception` function keyword.
```cpl
val ex: Exception = Exception("error!");
```
They have the following implementation:
```wasm
;; precursor to the `Exception` class
(type $Exception (sub final $Object (struct
	;; --- inherited ---
	;; unique id for hashing
	(field $id i64)

	;; --- own ---
	;; the error message
	(field $message (ref $String))
)))
```
The `Exception` keyword will be a “support class keyword” in the Counterpoint language. Eventually its class and type will be built in to the core library. It has the following signatures:
```cpl
declare class Exception {
	%%% Construct a new Exception object with a default message. %%%
	new ();
	%%% Construct a new Exception object with the given message. %%%
	new (message: str);
}
```

In Counterpoint, Exception objects are “falsy”, like `null`, `false`, `None` instances, and `Fail` instances.
```cpl
claim ex: Exception;
!!ex == false;
% And of course, all falsy objects are empty:
?ex == true;
```

## Throw Statements
Exceptions may be **thrown** by functions. This introduces a new `throw` keyword and statement. When an object is thrown, the throwing function “completes abruptly”: it does not finish execution, but instead sends the thrown object up to its caller, which in turn will do the same, until the top of the program is reached, at which point it will crash. There is no catching Exceptions or Exception handling. Functions that *definitely always throw* may be annotated with a return type of `nothing`, the Bottom Type.
```cpl
func throwException(message: str): nothing {
	val err: Exception = Exception(message);
	throw err;
}
```
Only Exception objects may be thrown. Attempting to throw a non-exception results in type error.
```cpl
func throwMessage(message: str): nothing {
	throw message; %> TypeError: Expression of type `str` is not assignable to type `Exception`.
}
```

Since Exceptions cannot be caught, we should only throw them in truly unrecoverable situations. For “exceptional circumstances”, you should *return* the Exception instead.
```cpl
func sqrt(x: float): float | Exception
	=> if x < 0.0
		then Exception("Argument must not be negative.")
		else x ^ 0.5;
```
When the function has a return type `float | Exception`, the caller may use that information to address the returned value how it sees fit, rather than being forced to deal with an Exception they didn’t ask for. For example:
```cpl
val valueOrEx: float | Exception = sqrt(x);
if !valueOrEx then {
	% `valueOrEx` is guaranteed to be an exception, since all floats, including 0.0, are “truthy”
	% do something with the exception
} else {
	% `valueOrEx` is guaranteed to be a float
	% do something with the float
};
```
That said, a more robust approach would be to return the built-in `Result` type (#100) and use pattern-matching.
```cpl
func sqrt(x: float): float!
	=> if x < 0.0
		then Fail("Argument must not be negative.")
		else Ok(x ^ 0.5);
val result: float! = sqrt(x);
when result is
	Fail             then """do something with {{ result.reason }}""",
	Ok               then """{{ result~! }} is a floating-point value.""",
	Ok[float](val x) then """{{ x }} is a floating-point value.""",
else null;
```

# Specification

## Lexicon
```diff
Keyword :::=
	// statement
+		| "throw"
;
```

## Syntax
```diff
 StatementBreak          ::= ("break" | "skip")                                      ";";
 StatementReturn <Break> ::=  "return"          Expression<+Block><?Break><+Return>? ";";
+StatementThrow  <Break> ::=  "throw"           Expression<+Block><?Break><+Return>  ";";

Statement<Break, Return> ::=
	| Declaration
	| StatementExpression<?Break><?Return>
	| StatementConditional<∓Unless><?Break><?Return>
	| StatementLoop<Return>
	| StatementIteration<Return>
	| <Break+> StatementBreak
	| <Return+>StatementReturn<?Break>
+	| StatementThrow<?Break>
;
```

## Semantics
```diff
SemanticStatement =:=
	| SemanticDeclaration
	| SemanticStatementExpression
	| SemanticStatementClaim
	| SemanticStatementReassignment
	| SemanticStatementConditional
	| SemanticStatementLoop
	| SemanticStatementIteration
	| SemanticStatementBreak
	| SemanticStatementReturn
;

-SemanticStatementReturn
+SemanticStatementReturn[throw=false]
	::= SemanticExpression?;
+SemanticStatementReturn[throw=true]
+	::= SemanticExpression;
```

## Decorate
```diff
Decorate(StatementReturn<Break> ::= "return" ";") -> SemanticStatementReturn
-	:= (SemanticStatementReturn);
+	:= (SemanticStatementReturn[throw=false]);
Decorate(StatementReturn<Break> ::= "return" Expression<+Block><?Break><+Return> ";") -> SemanticStatementReturn
-	:= (SemanticStatementReturn Decorate(Expression<+Block><?Break><+Return>));
+	:= (SemanticStatementReturn[throw=false] Decorate(Expression<+Block><?Break><+Return>));

+Decorate(StatementThrow<Break> ::= "throw" Expression<+Block><?Break><+Return> ";") -> SemanticStatementReturn
+	:= (SemanticStatementReturn[throw=true] Decorate(Expression<+Block><?Break><+Return>));

Decorate(Statement<Break, Return> ::= StatementReturn<?Break>) -> SemanticStatementReturn
	:= Decorate(StatementReturn<?Break>);
+Decorate(Statement<Break, Return> ::= StatementThrow<?Break>) -> SemanticStatementReturn
+	:= Decorate(StatementThrow<?Break>);
```

## TypeCheck
```diff
-Nil! TypeCheck(SemanticStatementReturn ::= ()) :=
+Nil! TypeCheck(SemanticStatementReturn[throw=false] ::= ()) :=
	1. *Let* `parent` be the result of walking up the AST and finding a node
		that is either a SemanticExpressionFunction or a SemanticDeclarationFunction.
	2. *If* `parent` is set:
		1. *Let* `return_node` be an item `node` in `parent` such that `item` is a SemanticType.
		2. *If* `return_node` is set:
			1. *Let* `return_type` be *Unwrap:* `TypeValueOf(return_node)`.
			2. *Throw:* a new Error """A function with return type \`{{ return_type.value }}\` does not return a value."""
		3. *Else:*
			1. *Note:* The function is a void function and the return statement has no expression.
			2. *Return.*
	3. *Else:*
		1. *Return.*
;
-Nil! TypeCheck(SemanticStatementReturn ::= SemanticExpression) :=
+Nil! TypeCheck(SemanticStatementReturn[throw=false] ::= SemanticExpression) :=
	1. *Let* `parent` be the result of walking up the AST and finding a node
		that is either a SemanticExpressionFunction or a SemanticDeclarationFunction.
	2. *If* `parent` is set:
		1. *Let* `return_node` be an item `node` in `parent` such that `item` is a SemanticType.
		2. *If* `return_node` is set:
			1. *Let* `return_type` be *Unwrap:* `TypeValueOf(return_node)`.
			2. *Return:* `TypeCheckAssign(SemanticExpression, return_type.value)`.
		3. *Else:*
			1. *Throw:* a new Error """Expression \`{{ SemanticExpression }}\` is returned from a void function."""
	3. *Else:*
		1. *Return.*
;
+Nil! TypeCheck(SemanticStatementReturn[throw=true] ::= SemanticExpression)
+	:= TypeCheckAssign(SemanticExpression, Exception);
```

## Build
```diff
-Build(SemanticStatementReturn ::= ()) := [ (GOTO "caller") ];
+Build(SemanticStatementReturn[throw=false] ::= ()) := [ (GOTO "caller") ];
-Build(SemanticStatementReturn ::= SemanticExpression) := [
+Build(SemanticStatementReturn[throw=false] ::= SemanticExpression) := [
	(RETURN `Build(SemanticExpression)`)
	(GOTO "caller")
];
+Build(SemanticStatementReturn[throw=true] ::= SemanticExpression) := [
+	// TODO: implement
+];
```

## ToBoolean
```diff
Boolean ToBoolean(Value value) :=
	1. *If* `value` is an instance of `Null`:
		1. *Return:* `false`.
	2. *If* `value` is an instance of `Boolean`:
		1. *Return:* `value`.
+	3. *If* `value` is an instance of `Exception`:
+		1. *Return:* `false`.
	4. *Return:* `true`.
```
