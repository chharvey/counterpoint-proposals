The `while` loop is a control flow statement that repeatedly executes based on a condition, and the `for` loop is a control flow statement that repeatedly executes over a range of integers.

Syntax:
```diff
+StatementLoop
+	::= (("while" | "until") Expression & "do" Block) ";";

+StatementIteration
+	::= "for" (
+		| IDENTIFIER          "from" Expression "to" Expression ("by" Expression)?
+		| IDENTIFIER ":" Type "of" Expression
+	) "do" Block ";";

Statement ::=
	| StatementExpression
	| StatementConditional
+	| StatementLoop
+	| StatementIteration
	| Declaration
;
```
Semantics:
```diff
SemanticCondition ::= SemanticExpression;
+SemanticStart    ::= SemanticExpression;
+SemanticEnd      ::= SemanticExpression;
+SemanticDelta    ::= SemanticExpression;
+SemanticIterable ::= SemanticExpression;

SemanticConsequent  ::= SemanticBlock;
SemanticAlternative ::= SemanticBlock | SemanticConditional;

SemanticConditional
	::= SemanticCondition SemanticConsequent SemanticAlternative;

+SemanticLoop[dofirst: Boolean]
+	::= SemanticCondition SemanticBlock;

+SemanticIteration ::=
+	| SemanticVariable SemanticStart SemanticEnd SemanticDelta SemanticBlock
+	| SemanticVariable SemanticType SemanticIterable           SemanticBlock
+;

SemanticStatement =:=
	| SemanticStatementExpression
	| SemanticConditional
+	| SemanticLoop
+	| SemanticIteration
	| SemanticDeclaration
;
```
Decorate:
```diff
+Decorate(StatementLoop ::= "while" Expression "do" Block ";") -> SemanticLoop
+	:= (SemanticLoop[dofirst=false]
+		(SemanticCondition Decorate(Expression))
+		Decorate(Block)
+	);
+Decorate(StatementLoop ::= "do" Block "while" Expression ";") -> SemanticLoop
+	:= (SemanticLoop[dofirst=true]
+		(SemanticCondition Decorate(Expression))
+		Decorate(Block)
+	);
+Decorate(StatementLoop ::= "until" Expression "do" Block ";") -> SemanticLoop
+	:= (SemanticLoop[dofirst=false]
+		(SemanticCondition
+			(SemanticOperation[operator=NOT] Decorate(Expression))
+		)
+		Decorate(Block)
+	);
+Decorate(StatementLoop ::= "do" Block "until" Expression ";") -> SemanticLoop
+	:= (SemanticLoop[dofirst=true]
+		(SemanticCondition
+			(SemanticOperation[operator=NOT] Decorate(Expression))
+		)
+		Decorate(Block)
+	);

+Decorate(StatementIteration ::= "for" IDENTIFIER "from" Expression__0 "to" Expression__1 "do" Block ";") -> SemanticIteration
+	:= (SemanticIteration
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		(SemanticStart Decorate(Expression__0))
+		(SemanticEnd   Decorate(Expression__1))
+		(SemanticDelta (SemanticConstant[value=Integer(1)]))
+		Decorate(Block)
+	);
+Decorate(StatementIteration ::= "for" IDENTIFIER "from" Expression__0 "to" Expression__1 "by" Expression__2 "do" Block ";") -> SemanticIteration
+	:= (SemanticIteration
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		(SemanticStart    Decorate(Expression__0))
+		(SemanticEnd      Decorate(Expression__1))
+		(SemanticDelta    Decorate(Expression__2))
+		Decorate(Block)
+	);
+Decorate(StatementIteration ::= "for" IDENTIFIER ":" Type "of" Expression "do" Block ";") -> SemanticIteration
+	:= (SemanticIteration
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		(SemanticType     Decorate(Type))
+		(SemanticIterable Decorate(Expression))
+		Decorate(Block)
+	);

+Decorate(Statement ::= StatementLoop) -> SemanticLoop
+	:= Decorate(StatementLoop);
+Decorate(Statement ::= StatementIteration) -> SemanticIteration
+	:= Decorate(StatementIteration);
```
Sugar:
```cp
until condition do { 0; };   % sugar for `while !condition do { 0; };`
do { 1; } until condition;   % sugar for `do { 1; } while !condition;`
for i from a to b do { 2; }; % sugar for `for i from a to b by 1 do { 2; };`
```

# Runtime Behavior
For `while-do` loops, the **condition** will be evaluated, and if it is *true* or a “truthy” value, the **body** will execute. These steps will repeat until the condition is *false* or “falsy”.

For `do-while` loops, the order is reversed: The body is executed for the first time, and then the condition is evaluated. The steps repeat until the condition is falsy.

`until-do`/`do-until` loops are syntax sugar for `while-do`/`do-while` loops respectively, with the condition logically negated.

There are two types of `for` loops: `for-from-to` loops (which are referred to as regular `for` loops), and `for-of` loops.

For (regular) `for` loops, the **index** will range from **start** to **end**, *not including the end*, incrementing or decrementing by **delta** (depending on whether the delta is positive or negative). For example, `for i from 4 to 10 by 2` means *i* will be 4, 6, and then 8 within the block. If no delta is specified, it defaults to 1. For each iteration, the **body** will execute until the index is no longer in range.

`for-of` loops iterate over indexed data types, such as tuples and lists, by index. For example, `for item: T of list` iterates over all of `list`’s items, declaring the variable `item` as type `T` and scoped to the block. The variable `item` is reassigned to the next item in `list` on each iteration. Unindexed data types (records, hashes, sets, and maps) cannot be directly iterated over, but a list of their entries may be. In the future, `for-of` loops will be able to iterate over generators as well.

# Compiler Short-Circuiting (non-normative preview)
The compiler will have an option to short-circuit loop and iteration statements. For `while-do` loops, if the compiler can determine the condition is falsy, the body will not be compiled. The same holds for `for` loops, if the compiler can determine the index will never be within range, and for `for-of` loops, if the iterable is empty. There is no compiler optimization for `do-while` loops.
