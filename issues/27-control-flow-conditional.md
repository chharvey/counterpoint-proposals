The `if` statement is a control flow statement that executes based on a condition. The `unless` statement is the negation of `if`.

Syntax:
```diff
+StatementConditional
+	::= ("if" | "unless") Expression "then" Block (";" | "else" (Block ";" | StatementConditional));

Statement ::=
	| StatementExpression
+	| StatementConditional
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
+Decorate(StatementConditional ::= "if" Expression "then" Block ";") -> SemanticConditional
+	:= (SemanticConditional
+		(SemanticCondition  Decorate(Expression))
+		(SemanticConsequent Decorate(Block))
+	);
+Decorate(StatementConditional ::= "if" Expression "then" Block__0 "else" Block__1 ";") -> SemanticConditional
+	:= (SemanticConditional
+		(SemanticCondition   Decorate(Expression))
+		(SemanticConsequent  Decorate(Block__0))
+		(SemanticAlternative Decorate(Block__1))
+	);
+Decorate(StatementConditional ::= "if" Expression "then" Block "else" StatementConditional) -> SemanticConditional
+	:= (SemanticConditional
+		(SemanticCondition   Decorate(Expression))
+		(SemanticConsequent  Decorate(Block))
+		(SemanticAlternative Decorate(StatementConditional))
+	);
+Decorate(StatementConditional ::= "unless" Expression "then" Block ";") -> SemanticConditional
+	:= (SemanticConditional
+		(SemanticCondition
+			(SemanticOperation[operator=NOT] Decorate(Expression))
+		)
+		(SemanticConsequent Decorate(Block))
+	);
+Decorate(StatementConditional ::= "unless" Expression "then" Block__0 "else" Block__1 ";") -> SemanticConditional
+	:= (SemanticConditional
+		(SemanticCondition
+			(SemanticOperation[operator=NOT] Decorate(Expression))
+		)
+		(SemanticConsequent  Decorate(Block__0))
+		(SemanticAlternative Decorate(Block__1))
+	);
+Decorate(StatementConditional ::= "unless" Expression "then" Block "else" StatementConditional) -> SemanticConditional
+	:= (SemanticConditional
+		(SemanticCondition
+			(SemanticOperation[operator=NOT] Decorate(Expression))
+		)
+		(SemanticConsequent  Decorate(Block))
+		(SemanticAlternative Decorate(StatementConditional))
+	);

+Decorate(Statement ::= StatementConditional) -> SemanticConditional
+	:= Decorate(StatementConditional);
```
Sugar:
```cp
if condition then { 1; };                 % sugar for `if  condition then { 1; } else {;};`
unless condition then { 1; } else { 2; }; % sugar for `if !condition then { 1; } else { 2; };`
```

Runtime behavior: A conditional `if` statement has a **condition** expression, a **consequent** block (“then” branch), and an optional **alternative** block (“else” branch). If the condition evaluates to true or a “truthy” value, the consequent is executed; else, the alternative is executed (or if the alternative doesn’t exist, nothing is executed). An `unless` statement is syntax sugar for an `if` statement with the condition logically negated.

Short-circuiting: At runtime, exactly one of the consequent or the alternative will be executed, and the branch that is not produced will not even be evaluated (for example, if it contains a procedure call, the procedure will not be called). Additionally, there are plans to include a compiler option that will skip not only evaluation but *compilation* of that branch, but only if the compiler can determine which branch will be executed based on the condition.
