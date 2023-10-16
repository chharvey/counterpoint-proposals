The `if` statement is a control flow statement that executes based on a condition. The `unless` statement is the negation of `if`.

Syntax:
```diff
+StatementConditional<If> ::=
+	. <If->"unless" . <If+>"if" Expression
+	. "then" Block
+	. (
+		| <If+>("else" Block)? ";"
+		| <If+>("else" StatementConditional<?If>)
+	)
+;

Statement ::=
	| StatementExpression
+	| StatementConditional<±If>
	| Declaration
;
```

Semantics:
```diff
+SemanticCondition
+	::= SemanticExpression;

+SemanticConsequent  ::= SemanticBlock;
+SemanticAlternative ::= SemanticBlock | SemanticConditional;

+SemanticConditional
+	::= SemanticCondition SemanticConsequent SemanticAlternative?;

SemanticStatement =:=
	| SemanticStatementExpression
+	| SemanticConditional
	| SemanticDeclaration
;
```

Decorate:
```diff
+Decorate(StatementConditional<+If> ::= "if" Expression "then" Block ";") -> SemanticConditional
+	:= (SemanticConditional
+		(SemanticCondition  Decorate(Expression))
+		(SemanticConsequent Decorate(Block))
+	);
+Decorate(StatementConditional<+If> ::= "if" Expression "then" Block__0 "else" Block__1 ";") -> SemanticConditional
+	:= (SemanticConditional
+		(SemanticCondition   Decorate(Expression))
+		(SemanticConsequent  Decorate(Block__0))
+		(SemanticAlternative Decorate(Block__1))
+	);
+Decorate(StatementConditional<+If> ::= "if" Expression "then" Block "else" StatementConditional<+If>) -> SemanticConditional
+	:= (SemanticConditional
+		(SemanticCondition   Decorate(Expression))
+		(SemanticConsequent  Decorate(Block))
+		(SemanticAlternative Decorate(StatementConditional))
+	);
+Decorate(StatementConditional<-If> ::= "unless" Expression "then" Block ";") -> SemanticConditional
+	:= (SemanticConditional
+		(SemanticCondition
+			(SemanticOperation[operator=NOT] Decorate(Expression))
+		)
+		(SemanticConsequent Decorate(Block))
+	);

+Decorate(Statement ::= StatementConditional<±If>) -> SemanticConditional
+	:= Decorate(StatementConditional<±If>);
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

Runtime behavior: A conditional `if` statement has a **condition** expression, a **consequent** block (“then” branch), and an optional **alternative** block (“else” branch). If the condition evaluates to true or a “truthy” value, the consequent is executed; else, the alternative is executed (or if the alternative doesn’t exist, nothing is executed). An `unless` statement is syntax sugar for an `if` statement with the condition logically negated. An `unless` statement cannot have an alternative block.

Short-circuiting: At runtime, exactly one of the consequent or the alternative will be executed, and the branch that is not produced will not even be evaluated (for example, if it contains a procedure call, the procedure will not be called). Additionally, there are plans to include a compiler option that will skip not only evaluation but *compilation* of that branch, but only if the compiler can determine which branch will be executed based on the condition.
