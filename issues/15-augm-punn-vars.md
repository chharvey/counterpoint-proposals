New assignment statements and update statements are syntactic sugar for special cases of reassignment.

# Description

## Augment Assignment
In addition to `=`, new (binary) augment operators are added, as well as a few (unary) update operators. These new operators are only syntax sugar; they do not add new semantics. The examples below informally explain the formal semantics.

```
a  ^= b; % sugar for `a = a  ^ b;`
a  *= b; % sugar for `a = a  * b;`
a  /= b; % sugar for `a = a  / b;`
a  += b; % sugar for `a = a  + b;`
a  -= b; % sugar for `a = a  - b;`
a &&= b; % sugar for `a = a && b;`
a !&= b; % sugar for `a = a !& b;`
a ||= b; % sugar for `a = a || b;`
a !|= b; % sugar for `a = a !| b;`

a++; % sugar for `a += 1;`
a--; % sugar for `a -= 1;`
a**; % sugar for `a *= 2;`
a//; % sugar for `a /= 2;`
```

Note on runtime short-circuiting: Because the boolean expressions `x || y`, `x !| y`, `x && y`, and `x !& y` short-circuit (see #22), the boolean augment operators `||=`, `!|=`, `&&=`, and `!&=` short-circuit in the same way. For example, if `a` is “truthy” and `‹b›` is some procedure (such as a function call), then the statement `a ||= ‹b›;` will not execute `‹b›`.

However, unlike some languages, the reassignment is never short-circuited and will always occur. Thus in the case of `a ||= ‹b›;`, if `a` is “truthy”, then `a` will be reassigned to the value of `a`. This is consistent with operations such as `x += 0;` where `x` is still reassigned to the value of `x`. The reassignment should only negligibly impact performance.

## Punning
Using the symbol `$` instead of an assigned expression is equivalent to repeating the assignee.
```
a = $; % sugar for `a = a;`
```
This isn’t particularly useful on its own, but it can be used with augment operators.
```
a  ^= $; % sugar for `a  ^= a;`
a  *= $; % sugar for `a  *= a;`
a  /= $; % sugar for `a  /= a;`
a  += $; % sugar for `a  += a;`
a  -= $; % sugar for `a  -= a;`
a &&= $; % sugar for `a &&= a;`
a !&= $; % sugar for `a !&= a;`
a ||= $; % sugar for `a ||= a;`
a !|= $; % sugar for `a !|= a;`
```

# Specification

## Lexical Grammar
```diff
Punctuator :::=
+	// constant
+		| "$"
	// statement
+		| "^="  | "*="  | "/="  | "+="  | "-=" |
+		| "&&=" | "!&=" | "||=" | "!|="
+		| "++"  | "--"  | "**"  | "//"
;
```

## Syntactic Grammar
```diff
+AugmentOperator ::= "^="  | "*="  | "/=" | "+=" | "-=" | "&&=" | "||=";
+AugmentNegate   ::= "!&=" | "!|=";
+UpdateOperator  ::= "++"  | "--"  | "**" | "//";

StatementAssignment
-	::= Assignee "=" Expression ";"
+	::= Assignee (("=" | AugmentOperator | AugmentNegate) ("$" | Expression) | UpdateOperator) ";"
;
```

## Decorate
```diff
+Decorate(StatementAssignment ::= Assignee "=" "$") -> SemanticAssignment
+	:= (SemanticAssignment
+		Decorate(Assignee)
+		Decorate(Assignee)
+	);
+Decorate(StatementAssignment ::= Assignee AugmentOperator "$") -> SemanticAssignment
+	:= (SemanticAssignment
+		Decorate(Assignee)
+		(SemanticOperation[operator=Decorate(AugmentOperator)]
+			Decorate(Assignee)
+			Decorate(Assignee)
+		)
+	);
+Decorate(StatementAssignment ::= Assignee AugmentNegate "$") -> SemanticAssignment
+	:= (SemanticAssignment
+		Decorate(Assignee)
+		(SemanticOperation[operator=NOT]
+			(SemanticOperation[operator=Decorate(AugmentNegate)]
+				Decorate(Assignee)
+				Decorate(Assignee)
+			)
+		)
+	);
Decorate(StatementAssignment ::= Assignee "=" Expression) -> SemanticAssignment
	:= (SemanticAssignment
		Decorate(Assignee)
		Decorate(Expression)
	);
+Decorate(StatementAssignment ::= Assignee AugmentOperator Expression) -> SemanticAssignment
+	:= (SemanticAssignment
+		Decorate(Assignee)
+		(SemanticOperation[operator=Decorate(AugmentOperator)]
+			Decorate(Assignee)
+			Decorate(Expression)
+		)
+	);
+Decorate(StatementAssignment ::= Assignee AugmentNegate Expression) -> SemanticAssignment
+	:= (SemanticAssignment
+		Decorate(Assignee)
+		(SemanticOperation[operator=NOT]
+			(SemanticOperation[operator=Decorate(AugmentNegate)]
+				Decorate(Assignee)
+				Decorate(Expression)
+			)
+		)
+	);
+Decorate(StatementAssignment ::= Assignee UpdateOperator) -> SemanticAssignment
+	:= (SemanticAssignment
+		Decorate(Assignee)
+		(SemanticOperation[operator=Decorate(UpdateOperator)]
+			Decorate(Assignee)
+			(SemanticConstant[value=DefaultOperand(UpdateOperator)])
+		)
+	);

+Decorate(AugmentOperator ::= "^=")  -> Operator := EXP;
+Decorate(AugmentOperator ::= "*=")  -> Operator := MUL;
+Decorate(AugmentOperator ::= "/=")  -> Operator := DIV;
+Decorate(AugmentOperator ::= "+=")  -> Operator := ADD;
+Decorate(AugmentOperator ::= "-=")  -> Operator := SUB;
+Decorate(AugmentOperator ::= "&&=") -> Operator := AND;
+Decorate(AugmentOperator ::= "||=") -> Operator := OR;
+Decorate(AugmentNegate   ::= "!&=") -> Operator := AND;
+Decorate(AugmentNegate   ::= "!|=") -> Operator := OR;

+Decorate(UpdateOperator ::= "++") -> Operator := ADD;
+Decorate(UpdateOperator ::= "--") -> Operator := SUB;
+Decorate(UpdateOperator ::= "**") -> Operator := MUL;
+Decorate(UpdateOperator ::= "//") -> Operator := DIV;

+DefaultOperand(UpdateOperator ::= "++") -> Integer := 1;
+DefaultOperand(UpdateOperator ::= "--") -> Integer := 1;
+DefaultOperand(UpdateOperator ::= "**") -> Integer := 2;
+DefaultOperand(UpdateOperator ::= "//") -> Integer := 2;
```
