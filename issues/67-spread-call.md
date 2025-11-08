Spread syntax injects tuples and records into function calls.

# Discussion
Functions have a static arity, that is, the number of parameters is known at compile-time. That number may be a range, if the function has optional parameters, but it’s still a static range. When a function is called, rather than providing arguments one by one, we can **spread** a tuple or record into the arguments list. Since the arity of the function is static, we can’t spread arrays or sets into its arguments. We also can’t spread mappings into function arguments.

We will use the following example to demonstrate:
```cp
function joinStrings(a: str, b: str, c: str, d: str, e: str, f: str): str
	=> '''{{ a }}{{ b }}{{ c }}{{ d }}{{ e }}{{ f }}''';
```

## Tuple Spread
Tuples may be spread into function call positional arguments with the `#` symbol. Spreading a tuple into a function call is equivalent to providing each positional argument individually.
```cp
let cde: [str, str, str] = ['c', 'd', 'e'];
let result: str = joinStrings.('a', 'b', #cde, 'f')~;
assert result == joinStrings.('a', 'b', cde.0, cde.1, cde.2 , 'f')~;
```

## Record Spread
Records may be spread into function call named arguments with the `##` symbol. Spreading a record into a function call is equivalent to providing each named argument individually. The evaluation order of named arguments in the spread record is that record’s insertion order.
```cp
let cde: [c: str, d: str, e: str] = [c= 'c', d= 'd', e= 'e'];
let result: str = joinStrings.(a= 'a', b= 'b', ##cde, f= 'f');
assert result == joinStrings.(a= 'a', b= 'b', c= cde.c, d= cde.d, e= cde.e , f= 'f')~;
```
Notice that spreading a record into named function arguments is similar to destructuring (#48), except that we don’t have to list out all the properties:
```cp
assert result == joinStrings(a= 'a', b= 'b', (c$, d$, e$)= cde, f= 'f')~;
```

## Limitations
Since tuples spread into positional arguments and records spread into named arguments, there are restrictions of the order these syntaxes can appear in a function call: All positional arguments and tuple spreads must come before all named arguments and record spreads. If any named argument or record spread appears before any positional argument or tuple spread in a function call, then it’s a syntax error.
```cp
my_function.(a= 42,     420)~;    % ParseError: positional argument cannot appear after named argument
my_function.(a= 42,     #[420])~; % ParseError: tuple spread        cannot appear after named argument
my_function.(##[a= 42], 420)~;    % ParseError: positional argument cannot appear after record spread
my_function.(##[a= 42], #[420])~; % ParseError: tuple spread        cannot appear after record spread
```
However, spread tuples and positional arguments may appear intermixed, as may spread records and named arguments.
```cp
my_function.(1, #[2, 3], 4, a= 5, ##[b= 6, c= 7], d= 8)~; % ok
```

# Specification

## Syntax
```diff
Label
	::= IDENTIFIER "=" Expression;

Arguments ::=
	| "(" ")"
-	| "(" ","?        Expression #                                  ","? ")"
-	| "(" ","? (      Expression # ",")?                    Label # ","? ")"
+	| "(" ","?  ("#"? Expression)#                                  ","? ")"
+	| "(" ","? (("#"? Expression)# ",")? ("##" Expression | Label)# ","? ")"
;
```

## Semantics
```diff
SemanticSpread[arity: 1 | 2 | 3]
	::= Expression;

SemanticCall ::=
	& SemanticExpression
-	& SemanticExpression*
-	& SemanticProperty*
+	& (SemanticExpression | SemanticSpread[arity: 1])*
+	& (SemanticProperty   | SemanticSpread[arity: 2])*
;
```

## Decorate
```diff
-Decorate(Arguments ::= "(" ","? Arguments__0__List ","? ")") -> Sequence<SemanticExpression>
+Decorate(Arguments ::= "(" ","? Arguments__0__List ","? ")") -> Sequence<SemanticExpression | SemanticSpread[arity: 1]>
	:= Decorate(Arguments__0__List);
-Decorate(Arguments ::= "(" ","? Arguments__1__List ","? ")") -> Sequence<SemanticProperty>
+Decorate(Arguments ::= "(" ","? Arguments__1__List ","? ")") -> Sequence<SemanticProperty | SemanticSpread[arity: 2]>
	:= Decorate(Arguments__1__List);
-Decorate(Arguments ::= "(" ","? Arguments__0__List "," Arguments__1__List ","? ")") -> Sequence<...Sequence<SemanticExpression>,                            ...Sequence<SemanticProperty>>
+Decorate(Arguments ::= "(" ","? Arguments__0__List "," Arguments__1__List ","? ")") -> Sequence<...Sequence<SemanticExpression | SemanticSpread[arity: 1]>, ...Sequence<SemanticProperty | SemanticSpread[arity: 2]>>
	:= [
		...Decorate(Arguments__0__List),
		...Decorate(Arguments__1__List),
	];

	Decorate(Arguments__0__List ::= Expression) -> Sequence<SemanticExpression>
		:= [Decorate(Expression)];
+	Decorate(Arguments__0__List ::= "#" Expression) -> Sequence<SemanticSpread[arity: 1]>
+		:= [(SemanticSpread[arity=1] Decorate(Expression))];
-	Decorate(Arguments__0__List ::= Arguments__0__List "," Expression) -> Sequence<SemanticExpression>
+	Decorate(Arguments__0__List ::= Arguments__0__List "," Expression) -> Sequence<SemanticExpression | SemanticSpread[arity: 1]>
		:= [
			...Decorate(Arguments__0__List),
			Decorate(Expression),
		];
+	Decorate(Arguments__0__List ::= Arguments__0__List "," "#" Expression) -> Sequence<SemanticExpression | SemanticSpread[arity: 1]>
+		:= [
+			...Decorate(Arguments__0__List),
+			(SemanticSpread[arity=1] Decorate(Expression)),
+		];
	Decorate(Arguments__1__List ::= Label) -> Sequence<SemanticProperty>
		:= [Decorate(Label)];
+	Decorate(Arguments__1__List ::= "##" Expression) -> Sequence<SemanticSpread[arity: 2]>
+		:= [(SemanticSpread[arity=2] Decorate(Expression))];
-	Decorate(Arguments__1__List ::= Arguments__1__List "," Label) -> Sequence<SemanticProperty>
+	Decorate(Arguments__1__List ::= Arguments__1__List "," Label) -> Sequence<SemanticProperty | SemanticSpread[arity: 2]>
		:= [
			...Decorate(Arguments__1__List),
			Decorate(Label),
		];
+	Decorate(Arguments__1__List ::= Arguments__1__List "," "##" Expression) -> Sequence<SemanticProperty | SemanticSpread[arity: 2]>
+		:= [
+			...Decorate(Arguments__1__List),
+			(SemanticSpread[arity=2] Decorate(Expression)),
+		];

-Decorate(FunctionCall ::= "." Arguments) -> Sequence<...Sequence<SemanticExpression>,                            ...Sequence<SemanticProperty>>
+Decorate(FunctionCall ::= "." Arguments) -> Sequence<...Sequence<SemanticExpression | SemanticSpread[arity: 1]>, ...Sequence<SemanticProperty | SemanticSpread[arity: 2]>>
	:= Decorate(Arguments);
```
