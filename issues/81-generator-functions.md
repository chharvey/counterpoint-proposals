Generator function syntax and `yield` statements.

# Discussion

## Generator Function Declarations
Generator functions are a syntax of functions that construct `Generator` objects. Rather than manually calling the `newGenerator.<T>()` function (see #80), we can write a function with a special syntax to construct a Generator and specify its yielded values.

Countdown example calling constructing function:
```cp
let countdown: Generator.<int> = newGenerator.<int>((yielder, returner, counter) {
	if counter == 0 then {
		yielder.(3);
	} else if counter == 1 then {
		yielder.(2);
	} else if counter == 2 then {
		yielder.(1);
	} else {
		returner.();
	};
});
```
Same example using generator function syntax:
```cp
function gen countdownFactory(): int {
	yield 3;
	yield 2;
	yield 1;
}
countdownFactory; %: gen () => int
let countdown: Generator.<int> = countdownFactory.();
```

Usage:
```cp
countdown.done;  %== false
countdown.count; %== 0

countdown++;     %== 3
countdown.done;  %== false
countdown.count; %== 1

countdown++;     %== 2
countdown.done;  %== false
countdown.count; %== 2

countdown++;     %== 1
countdown.done;  %== false
countdown.count; %== 3

countdown++;     %== null
countdown.done;  %== true
countdown.count; %== 3
```

Generator functions require the `gen` keyword in the header. Rather than `return` statements (which are not allowed), generator functions may contain `yield` statements, and they can have more than one. Each time “next” (`++`) is performed, the function resumes execution until the next `yield` statement (or until the function returns). The “return type” of the function is written as the type of yielded values. The function implicitly returns type `Generator.<T>`.

## Type Signatures
The type of a generator function has syntax `gen (‹params›) => ‹yield›`, where ‹params› are the parameter types and ‹yield› is the type of value yielded. It’s equivalent to the type signature `(‹params›) => Generator.<‹yield›>`.

## Generator Lambdas
Generator functions can be function expressions (“lambdas”). Generator lambdas are expressions and can be passed around as such.
```cp
let multOf: gen (int) => int = gen (base: int): int {
	for i from 1 to 0 do { % infinite loop
		yield i * base;
	};
};
```
Generator lambdas *cannot* have implicit returns. It’s a syntax error to write `gen () => 42` as an *expression*. (It is valid as a *type signature* though: a generator function that yields values of type `42`.)

# Specification

## Lexicon & Syntax
```diff
Keyword :::=
+	| "gen"
+	| "yield"
;

TypeFunction
-	::=        "(" ","? ParametersType<-Named, +Named> ")" "=>" Type;
+	::= "gen"? "(" ","? ParametersType<-Named, +Named> ")" "=>" Type;

-ExpressionFunction        ::=               "(" ","? ParametersFunction ","? ")" ":" Type         (StatementBlock<-Break><+Return><+Throw>         | "=>" Expression);
-DeclaredFunction          ::=               "(" ","? ParametersFunction ","? ")" ":" Type         (StatementBlock<-Break><+Return><+Throw>         | "=>" Expression ";");
+ExpressionFunction<Yield> ::= <Yield+>"gen" "(" ","? ParametersFunction ","? ")" ":" Type <Yield->(StatementBlock<-Break><+Return><+Throw><?Yield> | "=>" Expression)     <Yield+>StatementBlock<-Break><-Return><+Throw><?Yield>;
+DeclaredFunction<Yield>   ::=               "(" ","? ParametersFunction ","? ")" ":" Type <Yield->(StatementBlock<-Break><+Return><+Throw><?Yield> | "=>" Expression ";") <Yield+>StatementBlock<-Break><-Return><+Throw><?Yield>;

Expression ::=
	| ExpressionDisjunctive
	| ExpressionConditional
-	| ExpressionFunction
+	| ExpressionFunction<-Yield, +Yield>
;

StatementReturn ::= "return" Expression? ";";
StatementThrow  ::= "throw"  Expression? ";";
+StatementYield ::= "yield"  Expression  ";";

-Statement<Break, Return, Throw>        ::=
+Statement<Break, Return, Throw, Yield> ::=
	| Expression? ";"
	| StatementAssignment
	| StatementIf    <?Break>
	| StatementUnless<?Break>
	| StatementWhile
	| StatementUntil
	| StatementFor
	| <Break+>StatementBreak
	| <Break+>StatementContinue
	| <Return+>StatementReturn;
	| <Throw+>StatementThrow
+	| <Yield+>StatementYield
	| Declaration
;

-Block<Break, Return, Throw>        ::= "{" Statement<?Break><?Return><?Throw>+         "}";
+Block<Break, Return, Throw, Yield> ::= "{" Statement<?Break><?Return><?Throw><?Yield>+ "}";

-DeclarationFunction        ::= "func"               IDENTIFIER DeclaredFunction;
+DeclarationFunction<Yield> ::= "func" <Yield+>"gen" IDENTIFIER DeclaredFunction<?Yield>;

Declaration ::=
	| DeclarationVariable
	| DeclarationType
-	| DeclarationFunction
+	| DeclarationFunction<-Yield, Yield>
;
```
