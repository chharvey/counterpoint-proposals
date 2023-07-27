`break` and `continue` statements offer control over loop iterations.

Syntax:
```diff
-StatementConditional
+StatementConditional<Break>
-	::= ("if" | "unless") Expression "then" Block         (";" | "else" (Block         ";" | StatementConditional));
+	::= ("if" | "unless") Expression "then" Block<?Break> (";" | "else" (Block<?Break> ";" | StatementConditional<?Break>));

StatementLoop
-	::= (("while" | "until") Expression & "do" Block)         ";";
+	::= (("while" | "until") Expression & "do" Block<+Break>) ";";

StatementIteration
	::= "for" (
		| IDENTIFIER          "from" Expression "to" Expression ("by" Expression)?
		| IDENTIFIER ":" Type "of" Expression
-	) "do" Block         ";";
+	) "do" Block<+Break> ";";

+StatementBreak    ::= "break"    INTEGER? ";";
+StatementContinue ::= "continue" INTEGER? ";";

-Statement ::=
+Statement<Break> ::=
	| StatementExpression
-	| StatementConditional
+	| StatementConditional<?Break>
	| StatementLoop
	| StatementIteration
+	| <Break+>StatementBreak
+	| <Break+>StatementContinue
	| Declaration
;

-Block
+Block<Break> ::=
-	::= "{" Statement+         "}";
+	::= "{" Statement<?Break>+ "}";

Goal
-	::= #x02 Block?         #x03;
+	::= #x02 Block<-Break>? #x03;
```
Semantics:
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
Decorate:
```diff
-Decorate(StatementConditional        ::= "if" Expression "then" Block         ";") -> SemanticConditional
+Decorate(StatementConditional<Break> ::= "if" Expression "then" Block<?Break> ";") -> SemanticConditional
	:= (SemanticConditional
		(SemanticCondition  Decorate(Expression))
-		(SemanticConsequent Decorate(Block))
+		(SemanticConsequent Decorate(Block<?Break>))
	);
-Decorate(StatementConditional        ::= "if" Expression "then" Block__0         "else" Block__1         ";") -> SemanticConditional
+Decorate(StatementConditional<Break> ::= "if" Expression "then" Block__0<?Break> "else" Block__1<?Break> ";") -> SemanticConditional
	:= (SemanticConditional
		(SemanticCondition   Decorate(Expression))
-		(SemanticConsequent  Decorate(Block__0))
-		(SemanticAlternative Decorate(Block__1))
+		(SemanticConsequent  Decorate(Block__0<?Break>))
+		(SemanticAlternative Decorate(Block__1<?Break>))
	);
-Decorate(StatementConditional        ::= "if" Expression "then" Block         "else" StatementConditional        ) -> SemanticConditional
+Decorate(StatementConditional<Break> ::= "if" Expression "then" Block<?Break> "else" StatementConditional<?Break>) -> SemanticConditional
	:= (SemanticConditional
		(SemanticCondition   Decorate(Expression))
-		(SemanticConsequent  Decorate(Block))
-		(SemanticAlternative Decorate(StatementConditional))
+		(SemanticConsequent  Decorate(Block<?Break>))
+		(SemanticAlternative Decorate(StatementConditional<?Break>))
	);
-Decorate(StatementConditional        ::= "unless" Expression "then" Block         ";") -> SemanticConditional
+Decorate(StatementConditional<Break> ::= "unless" Expression "then" Block<?Break> ";") -> SemanticConditional
	:= (SemanticConditional
		(SemanticCondition
			(SemanticOperation[operator=NOT] Decorate(Expression))
		)
-		(SemanticConsequent  Decorate(StatementBlock))
+		(SemanticConsequent  Decorate(StatementBlock<?Break>))
	);
-Decorate(StatementConditional        ::= "unless" Expression "then" Block__0         "else" Block__1         ";") -> SemanticConditional
+Decorate(StatementConditional<Break> ::= "unless" Expression "then" Block__0<?Break> "else" Block__1<?Break> ";") -> SemanticConditional
	:= (SemanticConditional
		(SemanticCondition
			(SemanticOperation[operator=NOT] Decorate(Expression))
		)
-		(SemanticConsequent  Decorate(Block__0))
-		(SemanticAlternative Decorate(Block__1))
+		(SemanticConsequent  Decorate(Block__0<?Break>))
+		(SemanticAlternative Decorate(Block__1<?Break>))
	);
-Decorate(StatementConditional        ::= "unless" Expression "then" Block         "else" StatementConditional        ) -> SemanticConditional
+Decorate(StatementConditional<Break> ::= "unless" Expression "then" Block<?Break> "else" StatementConditional<?Break>) -> SemanticConditional
	:= (SemanticConditional
		(SemanticCondition
			(SemanticOperation[operator=NOT] Decorate(Expression))
		)
-		(SemanticConsequent  Decorate(Block))
-		(SemanticAlternative Decorate(StatementConditional))
+		(SemanticConsequent  Decorate(Block<?Break>))
+		(SemanticAlternative Decorate(StatementConditional<?Break>))
	);

-Decorate(StatementLoop ::= "while" Expression "do" Block         ";") -> SemanticLoop
+Decorate(StatementLoop ::= "while" Expression "do" Block<+Break> ";") -> SemanticLoop
	:= (SemanticLoop[dofirst=false]
		(SemanticCondition Decorate(Expression))
-		Decorate(Block)
+		Decorate(Block<+Break>)
	);
-Decorate(StatementLoop ::= "do" Block         "while" Expression ";") -> SemanticLoop
+Decorate(StatementLoop ::= "do" Block<+Break> "while" Expression ";") -> SemanticLoop
	:= (SemanticLoop[dofirst=true]
		(SemanticCondition Decorate(Expression))
-		Decorate(Block)
+		Decorate(Block<+Break>)
	);
-Decorate(StatementLoop ::= "until" Expression "do" Block         ";") -> SemanticLoop
+Decorate(StatementLoop ::= "until" Expression "do" Block<+Break> ";") -> SemanticLoop
	:= (SemanticLoop[dofirst=false]
		(SemanticCondition
			(SemanticOperation[operator=NOT] Decorate(Expression))
		)
-		Decorate(Block)
+		Decorate(Block<+Break>)
	);
-Decorate(StatementLoop ::= "do" Block         "until" Expression ";") -> SemanticLoop
+Decorate(StatementLoop ::= "do" Block<+Break> "until" Expression ";") -> SemanticLoop
	:= (SemanticLoop[dofirst=true]
		(SemanticCondition
			(SemanticOperation[operator=NOT] Decorate(Expression))
		)
-		Decorate(Block)
+		Decorate(Block<+Break>)
	);

-Decorate(StatementIteration ::= "for" IDENTIFIER "from" Expression__0 "to" Expression__1 "do" Block         ";") -> SemanticIteration
+Decorate(StatementIteration ::= "for" IDENTIFIER "from" Expression__0 "to" Expression__1 "do" Block<+Break> ";") -> SemanticIteration
	:= (SemanticIteration
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		(SemanticStart Decorate(Expression__0))
		(SemanticEnd   Decorate(Expression__1))
		(SemanticDelta (SemanticConstant[value=Integer(1)]))
-		Decorate(Block)
+		Decorate(Block<+Break>)
	);
-Decorate(StatementIteration ::= "for" IDENTIFIER "from" Expression__0 "to" Expression__1 "by" Expression__2 "do" Block         ";") -> SemanticIteration
+Decorate(StatementIteration ::= "for" IDENTIFIER "from" Expression__0 "to" Expression__1 "by" Expression__2 "do" Block<+Break> ";") -> SemanticIteration
	:= (SemanticIteration
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		(SemanticStart    Decorate(Expression__0))
		(SemanticEnd      Decorate(Expression__1))
		(SemanticDelta    Decorate(Expression__2))
-		Decorate(Block)
+		Decorate(Block<+Break>)
	);
-Decorate(StatementIteration ::= "for" IDENTIFIER ":" Type "of" Expression "do" Block         ";") -> SemanticIteration
+Decorate(StatementIteration ::= "for" IDENTIFIER ":" Type "of" Expression "do" Block<+Break> ";") -> SemanticIteration
	:= (SemanticIteration
		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
		(SemanticType     Decorate(Type))
		(SemanticIterable Decorate(Expression))
-		Decorate(Block)
+		Decorate(Block<+Break>)
	);

+Decorate(StatementBreak ::= "break" ";") -> SemanticBreak
+	:= (SemanticBreak[times=1]);
+Decorate(StatementBreak ::= "break" INTEGER ";")
+	:= (SemanticBreak[times=TokenWorth(INTEGER)]);

+Decorate(StatementContinue ::= "continue" ";") -> SemanticContinue
+	:= (SemanticContinue[times=1]);
+Decorate(StatementContinue ::= "continue" INTEGER ";") -> SemanticContinue
+	:= (SemanticContinue[times=TokenWorth(INTEGER)]);

-Decorate(Statement ::= StatementConditional) -> SemanticConditional
-	:= Decorate(StatementConditional);
+Decorate(Statement<Break> ::= StatementConditional<?Break>) -> SemanticConditional
+	:= Decorate(StatementConditional<?Break>);

+Decorate(Statement<+Break> ::= StatementBreak) -> SemanticBreak
+	:= Decorate(StatementBreak);
+Decorate(Statement<+Break> ::= StatementContinue) -> SemanticContinue
+	:= Decorate(StatementContinue);

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

Static Semantics: Semantic Error
```
StatementBreak ::= "continue" INTEGER ";"
StatementBreak ::= "break"    INTEGER ";"
```
It is a semantic error if `TokenWorth(INTEGER)` is negative.
