A Solid Script is a top-level file that contains a single statement block.

Currently the top-level Goal of the syntactic grammar must include a single Script (or nothing).

Syntax:
```diff
+StatementBlock
+	::= "{" Statement* "}";

Goal
-	::= #x02 Statement*      #x03;
+	::= #x02 StatementBlock? #x03;
```
Semantics:
```diff
+SemanticBlock
+	::= SemanticStatement*;

SemanticGoal
-	::= SemanticStatement*;
+	::= SemanticBlock?;
```
Decorate:
```diff
+Decorate(StatementBlock ::= "{" "}") -> SemanticBlock
+	:= (SemanticBlock);
+Decorate(StatementBlock ::= "{" Statement+ "}") -> SemanticBlock
+	:= (SemanticBlock
+		...ParseList(Statement, SemanticStatement)
+	);

Decorate(Goal ::= #x02 #x03) -> SemanticGoal
	:= (SemanticGoal);
-Decorate(Goal ::= #x02 Statement+     #x03) -> SemanticGoal
+Decorate(Goal ::= #x02 StatementBlock #x03) -> SemanticGoal
	:= (SemanticGoal
-		...ParseList(Statement, SemanticStatement)
+		Decorate(StatementBlock)
	);
```
