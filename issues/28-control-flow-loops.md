The `while` loop is a control flow statement that repeatedly executes based on a condition, and the `for` loop is a control flow statement that repeatedly executes over a range of integers or a list.

Syntax:
```diff
+StatementLoop
+	::= (("while" | "until") Expression<+Block> && "do" Block) ";";

+StatementIteration
+	::= "for" ("_" | IDENTIFIER) ":" Type "of" Expression<+Block> "do" Block ";";

Statement ::=
	| StatementExpression
	| StatementConditional<±Unless>
+	| StatementLoop
+	| StatementIteration
	| Declaration
;
```
Semantics:
```diff
SemanticIndex[index: RealNumber]
	::= ();

SemanticStatement =:=
	| SemanticStatementExpression
	| SemanticStatementConditional
+	| SemanticStatementLoop
+	| SemanticStatementIteration
	| SemanticDeclaration
;

+SemanticStatementLoop[dofirst: Boolean][until: Boolean]
+	::= SemanticExpression SemanticBlock;

+SemanticStatementIteration
+	::= SemanticVariable? SemanticType SemanticExpression SemanticBlock;
```
Decorate:
```diff
+Decorate(StatementLoop ::= "while" Expression<+Block> "do" Block ";") -> SemanticStatementLoop
+	:= (SemanticStatementLoop[dofirst=false][until=false]
+		Decorate(Expression<+Block>)
+		Decorate(Block)
+	);
+Decorate(StatementLoop ::= "do" Block "while" Expression<+Block> ";") -> SemanticStatementLoop
+	:= (SemanticStatementLoop[dofirst=true][until=false]
+		Decorate(Expression<+Block>)
+		Decorate(Block)
+	);
+Decorate(StatementLoop ::= "until" Expression<+Block> "do" Block ";") -> SemanticStatementLoop
+	:= (SemanticStatementLoop[dofirst=false][until=true]
+		Decorate(Expression<+Block>)
+		Decorate(Block)
+	);
+Decorate(StatementLoop ::= "do" Block "until" Expression<+Block> ";") -> SemanticStatementLoop
+	:= (SemanticStatementLoop[dofirst=true][until=true]
+		Decorate(Expression<+Block>)
+		Decorate(Block)
+	);

+Decorate(StatementIteration ::= "for" "_" ":" Type "of" Expression<+Block> "do" Block ";") -> SemanticStatementIteration
+	:= (SemanticStatementIteration
+		(SemanticType     Decorate(Type))
+		(SemanticIterable Decorate(Expression<+Block>))
+		Decorate(Block)
+	);
+Decorate(StatementIteration ::= "for" IDENTIFIER ":" Type "of" Expression<+Block> "do" Block ";") -> SemanticStatementIteration
+	:= (SemanticStatementIteration
+		(SemanticVariable[id=TokenWorth(IDENTIFIER)])
+		(SemanticType     Decorate(Type))
+		(SemanticIterable Decorate(Expression<+Block>))
+		Decorate(Block)
+	);

Decorate(Statement ::= StatementExpression) -> SemanticStatementExpression
	:= Decorate(StatementExpression);
Decorate(Statement ::= StatementConditional<±Unless>) -> SemanticStatementConditional
	:= Decorate(StatementConditional<±Unless>);
+Decorate(Statement ::= StatementLoop) -> SemanticStatementLoop
+	:= Decorate(StatementLoop);
+Decorate(Statement ::= StatementIteration) -> SemanticStatementIteration
+	:= Decorate(StatementIteration);
Decorate(Statement ::= Declaration) -> SemanticDeclaration
	:= Decorate(Declaration);
```
Effective Sugar:
```point
until condition do { 0; };   % `while !condition do { 0; };`
do { 1; } until condition;   % `do { 1; } while !condition;`
```

# Runtime Behavior
For `while-do` loops, the **condition** will be evaluated, and if it is *true*, the **body** will execute. These steps will repeat until the condition is *false*.

For `do-while` loops, the order is reversed: The body is executed for the first time, and then the condition is evaluated. The steps repeat until the condition is false.

`until-do`/`do-until` loops are syntax sugar for `while-do`/`do-while` loops respectively, with the condition logically negated.

There are two types of `for` loops: `for–of` loops, and `for–in` loops.

`for–of` loops iterate over dynamically indexed data types, such as lists and generators. For example, `for item: T of list` iterates over all of `list`’s items, declaring the variable `item` as type `T` and scoped to the block. The variable `item` is reassigned to the next item in `list` on each iteration. Static data types (tuples and records) and dynamically keyed data types (hashes, sets, and maps) cannot be directly iterated over, but a list of their entries may be.

# Compiler Short-Circuiting
The compiler will have an option to short-circuit loop and iteration statements. For `while-do` loops, if the compiler can determine the condition is false, the block body will not be compiled.

There is no similar Dead Code Elimination (#31) for `do-while` loops; however, if the `do` block is foldable (meaning it has no side-effects), then its output will be a *no-op* and only its `while` condition will be evaluated (and only once).
