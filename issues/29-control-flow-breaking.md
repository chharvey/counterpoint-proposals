`break` and `continue` statements offer control over loop iterations.

## Syntax
```diff
-ExpressionUnit<Block> ::=
+ExpressionUnit<Block, Break> ::=
	| IDENTIFIER
	| PrimitiveLiteral
-	| StringTemplate
-	| ExpressionGrouped
-	| TupleLiteral
-	| RecordLiteral
-	| ListLiteral
-	| DictLiteral
-	| SetLiteral
-	| MapLiteral
-	| <Block+>Block
+	| StringTemplate<?Break>
+	| ExpressionGrouped<?Break>
+	| TupleLiteral<?Break>
+	| RecordLiteral<?Break>
+	| ListLiteral<?Break>
+	| DictLiteral<?Break>
+	| SetLiteral<?Break>
+	| MapLiteral<?Break>
+	| <Block+>Block<?Break>
;

...

-Expression<Block> ::=
-	| ExpressionDisjunctive<?Block>
-	| ExpressionConditional
+Expression<Block, Break> ::=
+	| ExpressionDisjunctive<?Block><?Break>
+	| ExpressionConditional<?Break>
;

-StatementExpression        ::= Expression<+Block>?         ";";
+StatementExpression<Break> ::= Expression<+Block><?Break>? ";";

-StatementConditional<Unless> ::=
-	& <Unless->"if" <Unless+>"unless" Expression<+Block>
-	& "then" Block
+StatementConditional<Unless, Break> ::=
+	& <Unless->"if" <Unless+>"unless" Expression<+Block><?Break>
+	& "then" Block<?Break>
	& <Unless->(
-		| ("else" Block)? ";"
-		| "else" StatementConditional<?Unless>
+		| ("else" Block<?Break>)? ";"
+		| "else" StatementConditional<?Unless><?Break>
	)
	& <Unless+>";"
;

StatementLoop
-	::= (("while" | "until") Expression<+Block>         && "do" Block)         ";";
+	::= (("while" | "until") Expression<+Block><-Break> && "do" Block<+Break>) ";";

StatementIteration
-	::= "for" ("_" | IDENTIFIER) ":" Type "of" Expression<+Block>         "do" Block         ";";
+	::= "for" ("_" | IDENTIFIER) ":" Type "of" Expression<+Block><-Break> "do" Block<+Break> ";";

+StatementBreak ::= ("break" | "continue") ";";

-Statement ::=
+Statement<Break> ::=
-	| StatementExpression
-	| StatementConditional<∓Unless>
+	| StatementExpression<?Break>
+	| StatementConditional<∓Unless><?Break>
	| StatementLoop
	| StatementIteration
+	| <Break+>StatementBreak
	| Declaration
;

-Block        ::= "{" Statement+         "}";
+Block<Break> ::= "{" Statement<?Break>+ "}";

-DeclarationVariable ::=
+DeclarationVariable<Break> ::=
-	| "let" "var"? ("_" | IDENTIFIER) ":"  Type "=" Expression<+Block>         ";"
+	| "let" "var"? ("_" | IDENTIFIER) ":"  Type "=" Expression<+Block><?Break> ";"
	| "let" "var"  ("_" | IDENTIFIER) "?:" Type                                ";"
;

-DeclarationReassignment        ::= "set" Assignee         "=" Expression<+Block>         ";";
+DeclarationReassignment<Break> ::= "set" Assignee<?Break> "=" Expression<+Block><?Break> ";";

...

Goal
-	::= #x02 Block?         #x03;
+	::= #x02 Block<-Break>? #x03;
```

## Semantics
```diff
+SemanticStatementBreak[continue: Boolean]
	::= ();

SemanticStatement =:=
	| SemanticStatementExpression
	| SemanticConditional
	| SemanticLoop
	| SemanticIteration
+	| SemanticStatementBreak
	| SemanticDeclaration
;
```

## Decorate
Update all `Expression*`, `Statement*`, and `Declaration*` productions with the `<Break>` parameter.
```diff
+Decorate(StatementBreak ::= "break"    ";") -> SemanticStatementBreak := (SemanticStatementBreak[continue=false]);
+Decorate(StatementBreak ::= "continue" ";") -> SemanticStatementBreak := (SemanticStatementBreak[continue=true]);

-Decorate(Statement ::= StatementExpression) -> SemanticStatementExpression
-	:= Decorate(StatementExpression);
-Decorate(Statement ::= StatementConditional<∓Unless>) -> SemanticConditional
-	:= Decorate(StatementConditional<∓Unless>);
+Decorate(Statement<Break> ::= StatementExpression<?Break>) -> SemanticStatementExpression
+	:= Decorate(StatementExpression<?Break>);
+Decorate(Statement<Break> ::= StatementConditional<∓Unless><?Break>) -> SemanticConditional
+	:= Decorate(StatementConditional<∓Unless><?Break>);
+Decorate(Statement<Break> ::= StatementBreak) -> SemanticStatementBreak
+	:= Decorate(StatementBreak);

-Decorate(Block        ::= "{" Statement+         "}") -> SemanticBlock
+Decorate(Block<Break> ::= "{" Statement<?Break>+ "}") -> SemanticBlock
	:= (SemanticBlock
-		...ParseList(Statement,         SemanticStatement)
+		...ParseList(Statement<?Break>, SemanticStatement)
	);

Decorate(Goal ::= #x02 #x03) -> SemanticGoal
	:= (SemanticGoal);
-Decorate(Goal ::= #x02 Block         #x03) -> SemanticGoal
+Decorate(Goal ::= #x02 Block<-Break> #x03) -> SemanticGoal
	:= (SemanticGoal
-		Decorate(Block)
+		Decorate(Block<-Break>)
	);
```

## Discussion
Inside `while`/`until` and `for` loops, if a `continue` or `break` statement is encountered in the body, the rest of the body will cease to execute. If the statement is a `continue` statement, the current iteration will end and the next iteration will proceed. If it’s a `break` statement, the current iteration and any subsequent iterations will end — the loop will terminate altogether.
