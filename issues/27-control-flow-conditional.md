The `if` statement is a control flow statement that executes based on a condition. The `unless` statement is the negation of `if`.

Syntax:
```diff
+StatementConditional<Unless> ::=
+	& <Unless->"if" <Unless+>"unless" Expression
+	& "then" Block
+	& <Unless->(
+		| ("else" Block)? ";"
+		| "else" StatementConditional<?Unless>
+	)
+	& <Unless+>";"
+;

Statement ::=
	| StatementExpression
+	| StatementConditional<±Unless>
	| Declaration
;
```

Semantics:
```diff
SemanticStatement =:=
	| SemanticStatementExpression
+	| SemanticStatementConditional
	| SemanticDeclaration
;

+SemanticStatementConditional
+	::= SemanticExpression SemanticBlock (SemanticBlock | SemanticStatementConditional)?;
```

Decorate:
```diff
+Decorate(StatementConditional<-Unless> ::= "if" Expression "then" Block ";") -> SemanticStatementConditional
+	:= (SemanticStatementConditional[unlesss=false]
+		Decorate(Expression)
+		Decorate(Block)
+	);
+Decorate(StatementConditional<-Unless> ::= "if" Expression "then" Block__0 "else" Block__1 ";") -> SemanticStatementConditional
+	:= (SemanticStatementConditional[unlesss=false]
+		Decorate(Expression)
+		Decorate(Block__0)
+		Decorate(Block__1)
+	);
+Decorate(StatementConditional<-Unless> ::= "if" Expression "then" Block "else" StatementConditional<-Unless>) -> SemanticStatementConditional
+	:= (SemanticStatementConditional[unlesss=false]
+		Decorate(Expression)
+		Decorate(Block)
+		Decorate(StatementConditional<-Unless>)
+	);
+Decorate(StatementConditional<+Unless> ::= "unless" Expression "then" Block ";") -> SemanticStatementConditional
+	:= (SemanticStatementConditional[unlesss=true]
+		Decorate(Expression)
+		Decorate(Block)
+	);

+Decorate(Statement ::= StatementConditional<±Unless>) -> SemanticStatementConditional
+	:= Decorate(StatementConditional<±Unless>);
```

Sugar:
```cp
if condition then { 1; };     % sugar for `if  condition then { 1; } else {;};`
unless condition then { 1; }; % sugar for `if !condition then { 1; } else {;};`

if condition then { 1; } else if condition2 then { 2; } else { 3; }; %% sugar for
if condition then {
	1;
} else {
	if condition2 then {
		2;
	} else {
		3;
	};
};
%%
```

Runtime behavior: A conditional `if` statement has a **condition** expression, a **consequent** block (“then” branch), and an optional **alternative** block (“else” branch). The condition must be a boolean type, just like in the ternary conditional operation. If it evaluates to true, the consequent is executed; else, the alternative is executed (or if the alternative doesn’t exist, nothing is executed). An `unless` statement is syntax sugar for an `if` statement with the condition logically negated. An `unless` statement cannot have an alternative block.

Short-circuiting: At runtime, exactly one of the consequent or the alternative will be executed, and the branch that is not produced will not even be evaluated (for example, if it contains a procedure call, the procedure will not be called). Additionally, there are plans to include a compiler option that will skip not only evaluation but *compilation* of that branch, but only if the compiler can determine which branch will be executed based on the condition.
