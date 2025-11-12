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


-Statement ::=
+Statement<Break> ::=
-	| StatementExpression
-	| StatementConditional<∓Unless>
+	| StatementExpression<?Break>
+	| StatementConditional<∓Unless><?Break>
	| StatementLoop
	| StatementIteration
+	| <Break+>("break"    INTEGER? ";")
+	| <Break+>("continue" INTEGER? ";")
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

-DeclarationReassignment        ::= "set" Assignee "=" Expression<+Block>         ";";
+DeclarationReassignment<Break> ::= "set" Assignee "=" Expression<+Block><?Break> ";";

Goal
-	::= #x02 Block?         #x03;
+	::= #x02 Block<-Break>? #x03;
```

## Semantics
```diff
+SemanticBreak   [times: RealNumber] ::= ();
+SemanticContinue[times: RealNumber] ::= ();

SemanticStatement =:=
	| SemanticStatementExpression
	| SemanticConditional
	| SemanticLoop
	| SemanticIteration
+	| SemanticBreak
+	| SemanticContinue
	| SemanticDeclaration
;
```

### Semantic Error
```
SemanticBreak   [times: RealNumber] ::= ();
SemanticContinue[times: RealNumber] ::= ();
```
It is a semantic error if `[times]` is negative.

Note: Given an unsigned int type (#107), this semantic error may not be needed. In that case, the following syntax would be changed:
```diff
Statement<Break> ::=
# ...
-	| <Break+>("break"    INTEGER? ";")
-	| <Break+>("continue" INTEGER? ";")
+	| <Break+>("break"    NATURAL? ";")
+	| <Break+>("continue" NATURAL? ";")
# ...
;
```

## Decorate
Update all `Expression*`, `Statement*`, and `Declaration*` productions with the `<Break>` parameter.
```diff
-Decorate(Statement ::= StatementExpression) -> SemanticStatementExpression
-	:= Decorate(StatementExpression);
-Decorate(Statement ::= StatementConditional<∓Unless>) -> SemanticConditional
-	:= Decorate(StatementConditional<∓Unless>);
+Decorate(Statement<Break> ::= StatementExpression<?Break>) -> SemanticStatementExpression
+	:= Decorate(StatementExpression<?Break>);
+Decorate(Statement<Break> ::= StatementConditional<∓Unless><?Break>) -> SemanticConditional
+	:= Decorate(StatementConditional<∓Unless><?Break>);

+Decorate(Statement<Break> ::= <Break+>("break" ";")) -> SemanticBreak
+	:= (SemanticBreak[times=1]);
+Decorate(Statement<Break> ::= <Break+>("break" INTEGER ";")) -> SemanticBreak
+	:= (SemanticBreak[times=TokenWorth(INTEGER)]);
+Decorate(Statement<Break> ::= <Break+>("continue" ";")) -> SemanticContinue
+	:= (SemanticContinue[times=1]);
+Decorate(Statement<Break> ::= <Break+>("continue" INTEGER ";")) -> SemanticContinue
+	:= (SemanticContinue[times=TokenWorth(INTEGER)]);

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

If a non-negative integer follows the keyword `continue`, as in `continue 3;`, and the loop is nested in other loops, then that number, minus *1*, of innermost ancestor loops will be terminated, and then the surrounding loop will ‘continue’ (defined above). The statement `continue;` is a synonym of `continue 0;`.

Example of nested continuing:
```cp
while «expressionA» do {
	while «expressionB» do {
		while «expressionC» do {
			while «expressionD» do {
				continue 2;
				% this loop will break
			};
			% this loop will break
		};
		% this iteration will stop but this loop will continue
	};
	% this loop will not be affected
};
```

If a non-negative integer follows the keyword `break`, as in `break 3;`, and the loop is nested in other loops, that number of innermost ancestor loops will also be terminated. The statement `break;` is a synonym of `break 0;`.

Example of nested breaking:
```cp
while «expression» do {
	while «expression» do {
		while «expression» do {
			while «expression» do {
				break 2;
				% this loop will break
			};
			% this loop will break
		};
		% this loop will break
	};
	% this loop will not be affected
};
```
