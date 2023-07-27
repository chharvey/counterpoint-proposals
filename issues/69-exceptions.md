The Exception type is a new built-in type. It may be thrown by functions.

# Discussion

## Exception Type
In this version, the Exception type is a record type that has one property, `message`, which is a `str`. Once we get classes, `Exception` will be a class with its own constructor, but for now, “Exception objects” are constructed as records. They must have the following type:
```cp
type Exception = [message: str];
```
The `Exception` keyword will be a “support class keyword” in the Counterpoint language. Eventually its class and type will be built in to the core library.

In Counterpoint, Exception objects are “falsy”. They are the only objects in the entire language (besides `null` and `false`) that are falsy.
```cp
let ex: Exception = [message= 'error!'];
!ex == true;
% And of course, all falsy objects are empty:
?ex == true;
```

## Throw Statements
Exceptions may be **thrown** by functions. This introduces a new `throw` keyword and statement. When an object is thrown, the throwing function “completes abruptly”: it does not finish execution, but instead sends the thrown object up to its caller, which in turn will do the same, until the top of the program is reached, at which point it will crash. There is no catching Exceptions or Exception handling. Functions that *definitely always throw* may be annotated with a return type of `never`, since they never truly return.
```cp
func throwException(message: str): never {
	let err: Exception = [message$];
	throw err;
}
```
Only Exception objects may be thrown. Attempting to throw a non-exception results in type error.
```cp
func throwInt(n: int): never {
	throw n; %> TypeError: Expression of type `int` is not assignable to type `Exception`.
}
```

However, there is a shorthand syntax for throwing an Exception: using `throw` on a string. This doesn’t actually throw the string, but wraps it in an Exception type.
```cp
func throwHello(): never {
	throw 'hello'; % equivalent to `throw [message= 'hello'];`
}
```
An even shorter syntax is to use `throw` with no operand (throwing an Exception with an empty message). It’s not particularly helpful in production, but it can be ergonomic for debugging.
```cp
func throwEmpty(): never {
	throw; % equivalent to `throw [message= ''];`
}
```

Since Exceptions cannot be caught, we should only throw them in truly unrecoverable situations. For “exceptional circumstances”, the best practice is to *return* the Exception instead.
```cp
func sqrt(x: float): float | Exception
	=> if x < 0.0
		then [message= 'Argument must not be negative.']
		else x ^ 0.5;
```
When the function has a return type `float | Exception`, the caller may use that information to address the returned value how it sees fit, rather than being forced to deal with an Exception they didn’t ask for. For example:
```cp
let valueOrExcp: float | Exception = sqrt.(x)~~;
if !value then {
	% value is guaranteed to be an exception, since all floats, including 0.0, are “truthy”
	% do something with the exception
} else {
	% value is guaranteed to be a float
	% do something with the float
};
```

## Exception-ish Type Operator
The punctuator `!` (U+0021 EXCLAMATION MARK) is a postfix unary operator for “Exception-ish”, that is, union with Exception. Below are the new operators.
```cp
type IntOrNull      = int?; % int | null
type IntOrException = int!; % int | Exception
```
The new “Exception-ish” operator is a nice shorthand for function return types.
```cp
func sqrt(x: float): float!
	=> if x < 0.0
		then [message= 'Argument must not be negative.']
		else x ^ 0.5;
let valueOrExcp: float! = sqrt.(x)~~;
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
TypeUnarySymbol ::=
	| TypeUnit
	| TypeUnarySymbol "?"
+	| TypeUnarySymbol "!"
;

+StatementThrow
+	::= "throw" Expression? ";";

-Statement ::=
+Statement<Throw> ::=
	| Expression? ";"
	| Declaration
	| StatementAssignment
+	| <Throw+>StatementThrow
;

-ExpressionFunction ::= "(" ","? ParametersFunction ")" ":" Type (StatementBlock<-Break>         | ImplicitReturn);
-DeclaredFunction   ::= "(" ","? ParametersFunction ")" ":" Type (StatementBlock<-Break>         | ImplicitReturn ";");
+ExpressionFunction ::= "(" ","? ParametersFunction ")" ":" Type (StatementBlock<-Break><+Throw> | ImplicitReturn);
+DeclaredFunction   ::= "(" ","? ParametersFunction ")" ":" Type (StatementBlock<-Break><+Throw> | ImplicitReturn ";");
```

## Semantics
```diff
SemanticStatement =:=
	| SemanticStatementExpression
	| SemanticDeclaration
	| SemanticAssignment
+	| SemanticThrow
;

+SemanticThrow
+	::= SemanticExpression?;
```

## Decorate
```diff
Decorate(TypeUnarySymbol ::= TypeUnarySymbol "?") -> SemanticTypeOperation
	:= (SemanticTypeOperation[operator=ORNULL]
		Decorate(TypeUnarySymbol)
	);
+Decorate(TypeUnarySymbol ::= TypeUnarySymbol "!") -> SemanticTypeOperation
+	:= (SemanticTypeOperation[operator=OREXCEPTION]
+		Decorate(TypeUnarySymbol)
+	);

+Decorate(StatementThrow ::= "throw" ";") -> SemanticThrow
+	:= (SemanticThrow);
+Decorate(StatementThrow ::= "throw" Expression ";") -> SemanticThrow
+	:= (SemanticThrow Decorate(Expression));

+Decorate(Statement_Throw ::= StatementThrow) -> SemanticThrow
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
	4. *Return:* `true`.
```
