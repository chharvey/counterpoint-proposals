The Exception type is a new built-in type. It may be thrown by functions.

# Discussion

## Exception Type
In this version, the Exception type is a record type that has one property, `message`, which is a `str`. Once we get classes, `Exception` will be a class with its own constructor, but for now, “Exception objects” are constructed as records. They must have the following type:
```cpl
type Exception = (message: str);
```
The `Exception` keyword will be a “support class keyword” in the Counterpoint language. Eventually its class and type will be built in to the core library.

In Counterpoint, Exception objects are “falsy”. They are the only objects in the entire language (besides `null` and `false`) that are falsy.
```cpl
val ex: Exception = (message= "error!");
!!ex == false;
% And of course, all falsy objects are empty:
?ex == true;
```

## Throw Statements
Exceptions may be **thrown** by functions. This introduces a new `throw` keyword and statement. When an object is thrown, the throwing function “completes abruptly”: it does not finish execution, but instead sends the thrown object up to its caller, which in turn will do the same, until the top of the program is reached, at which point it will crash. There is no catching Exceptions or Exception handling. Functions that *definitely always throw* may be annotated with a return type of `nothing`, the Bottom Type.
```cpl
func throwException(message: str): nothing {
	val err: Exception = ($message);
	throw err;
}
```
Only Exception objects may be thrown. Attempting to throw a non-exception results in type error.
```cpl
func throwInt(n: int): nothing {
	throw n; %> TypeError: Expression of type `int` is not assignable to type `Exception`.
}
```

However, there is a shorthand syntax for throwing an Exception: using `throw` on a string. This doesn’t actually throw the string, but wraps it in an Exception type.
```cpl
func throwHello(): nothing {
	throw "hello"; % equivalent to `throw (message= "hello");`
}
```
An even shorter syntax is to use `throw` with no operand (throwing an Exception with an empty message). It’s not particularly helpful in production, but it can be ergonomic for debugging.
```cpl
func throwEmpty(): nothing {
	throw; % equivalent to `throw (message= "");`
}
```

Since Exceptions cannot be caught, we should only throw them in truly unrecoverable situations. For “exceptional circumstances”, the best practice is to *return* the Exception instead.
```cpl
func sqrt(x: float): float | Exception
	=> if x < 0.0
		then (message= "Argument must not be negative.")
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
	Fail              then """do something with {{ result.reason }}""",
	Ok                then """{{ result~! }} is a floating-point value.""",
	Ok.<float>(val x) then """{{ x }} is a floating-point value.""",
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
+StatementThrow<Break>
+	::= "throw" Expression<+Block><?Break><+Return>? ";";

-Statement<Break, Return> ::=
+Statement<Break, Return, Throw> ::=
	| Declaration
	| StatementExpression<?Break><?Return>
	| StatementConditional<∓Unless><?Break><?Return>
	| StatementLoop<Return>
	| StatementIteration<Return>
	| <Break+> StatementBreak
	| <Return+>StatementReturn<?Break>
+	| <Throw+>StatementThrow<?Break>
;

-DeclaredFunction   ::=     "(" ParametersFunction? ")" ":" ("void" | Type) (Block<-Break><+Return>         | "=>" Expression<+Block><-Break><+Return> ";");
-ExpressionFunction ::= "\" "(" ParametersFunction? ")" ":" ("void" | Type) (Block<-Break><+Return>         | "=>" Expression<+Block><-Break><+Return>);
+DeclaredFunction   ::=     "(" ParametersFunction? ")" ":" ("void" | Type) (Block<-Break><+Return><+Throw> | "=>" Expression<+Block><-Break><+Return> ";");
+ExpressionFunction ::= "\" "(" ParametersFunction? ")" ":" ("void" | Type) (Block<-Break><+Return><+Throw> | "=>" Expression<+Block><-Break><+Return>);
```

## Semantics
```diff
SemanticStatement =:=
	| SemanticDeclaration
	| SemanticStatementExpression
	| SemanticStatementConditional
	| SemanticLoop
	| SemanticIteration
	| SemanticBreak
	| SemanticContinue
	| SemanticReturn
+	| SemanticThrow
;

+SemanticThrow
+	::= SemanticExpression?;
```

## Decorate
```diff
+Decorate(StatementThrow ::= "throw" ";") -> SemanticThrow
+	:= (SemanticThrow);
+Decorate(StatementThrow ::= "throw" Expression ";") -> SemanticThrow
+	:= (SemanticThrow Decorate(Expression));

+Decorate(Statement<Throw> ::= <Throw+>StatementThrow) -> SemanticThrow
+	:= Decorate(StatementThrow);
```

## ToBoolean
```diff
Boolean ToBoolean(Object value) :=
	1. *If* `value` is an instance of `Null`:
		1. *Return:* `false`.
	2. *If* `value` is an instance of `Boolean`:
		1. *Return:* `value`.
+	3. *If* `value` is an instance of `Record` *and* `value` has a "message" key *and* `value.message` is an instance of `String`:
+		1. *Return:* `false`.
+		2. *Note:* This step is only temporary until the Exception class is created.
	4. *Return:* `true`.
```
