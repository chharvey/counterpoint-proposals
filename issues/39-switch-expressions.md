Switch Expressions compare (by identity `===`) an operand with a set of options return a result based on the corresponding option. It’s similar to a typical `switch` statement in other programming languages, except it’s an declarative expression (with a type and a value) rather than an imperative statement.

# Example
```cpl
val letter: str = switch 42.0
	case 24.0       -> "a"
	case 10.5 * 4.0 -> "b"
	case 10.5 | 4.0 -> "c"
default "d";
```

# Description
<table>
	<thead>
		<tr>
			<th>Precedence<br/><small>(1 is highest)</small></th>
			<th>Operator Name</th>
			<th>Arity</th>
			<th>Grouping</th>
			<th>Symbols</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<th>14</th>
			<td>Switch</td>
			<td>n-ary infix</td>
			<td>n/a</td>
			<td><code>switch …</code> (<code>case</code> (<code>… |</code>)* <code>… -> …</code>)* <code>default …</code></td>
		</tr>
	</tbody>
</table>

The `switch` expression compares the operand with a set of options and returns a corresponding result. Like the conditional expression, it short-circuits, meaning that once a result is found, it is produced without evaluating the rest of the expression. Specifically, the test is a comparison via identity (`===`). Each **case** contains one or more **antecedents** and one **consequent**. The result of the `switch` expression is the consequent that corresponds to the matched antecedent, or, if none is matched, then the default value.

```cpl
switch operand
	case option1 -> result1
	case option2 -> result2
	case option3 -> result3
default result4;
```
If `operand === option1`, the expression produces `result1`; if `operand === option2`, then `result2` is produced, and so on. If the value doesn’t equal any option, then `result4` (the **default**) is produced. The expression short-circuits, so it doesn’t evaluate any options or results after the first match. If several options are identical (e.g. if `option2 === option3`), then the first option to match in source order is selected.

Cases may have multiple pipe-separated antecedents.
```cpl
switch value
	case 10      -> result1
	case 21 | 22 -> a_lot_of_code_for_result2
	case 31 | 32 -> even_more_code_for_result3
default result4;
```
equivalent to:
```cpl
if value === 10                 then result1                    else
if value === 21 || value === 22 then a_lot_of_code_for_result2  else
if value === 31 || value === 32 then even_more_code_for_result3 else
result4;
```

Switch expressions may have 0 cases; if so, the default is automatically produced. The default is required for all switch expressions.
```cpl
switch 42 default 2.4; %== 2.4
```

# Specfication Changes

## Lexical Grammar
```diff
Keyword :::=
	// operator
+		| "switch"
+		| "case"
+		| "default"
;
```

## Syntactic Grammar
```diff
Property    ::=                          Word       "="  Expression;
-Case       ::=                          Expression "->" Expression;
+CaseMap    ::=                          Expression "->" Expression;
+CaseSwitch ::= "case" (Expression "|")* Expression "->" Expression;

-MapLiteral ::= "{" ","? Case#    ","? "}";
+MapLiteral ::= "{" ","? CaseMap# ","? "}";

+ExpressionSwitch
+	::= "switch" Expression CaseSwitch# "default" Expression;

Expression ::=
	| ExpressionDisjunctive
	| ExpressionConditional
+	| ExpressionSwitch
;
```

## Semantic Schema
```diff
-SemanticCase
+SemanticCaseMap
	::= SemanticExpression SemanticExpression;

+SemanticCaseSwitch
+	::= SemanticExpression+ SemanticExpression;

+SemanticExpressionOperation[operator: SWITCH]
+	::= SemanticExpression SemanticCaseSwitch* SemanticExpression;
```

## Decorate
```diff
-Decorate(Case    ::= Expression "->" Expression) -> SemanticCase
+Decorate(CaseMap ::= Expression "->" Expression) -> SemanticCaseMap
-	:= (SemanticCase
+	:= (SemanticCaseMap
		Decorate(Expression)
		Decorate(Expression)
	);

+Decorate(CaseSwitch ::= "case" Expression__0 "->" Expression__1) -> SemanticCaseSwitch
+	:= (SemanticCaseSwitch
+		Decorate(Expression__0)
+		Decorate(Expression__1)
+	);
+Decorate(CaseSwitch ::= "case" (Expression__0 "|")+ Expression__1 "->" Expression__2) -> SemanticCaseSwitch
+	:= (SemanticCaseSwitch
+		...ParseList(Expression__0, SemanticExpression)
+		Decorate(Expression__1)
+		Decorate(Expression__2)
+	);

+Decorate(ExpressionSwitch ::= "switch" Expression__0 "default" Expression__1) -> SemanticExpressionOperation
+	:= (SemanticExpressionOperation[operator=SWITCH]
+		Decorate(Expression__0)
+		Decorate(Expression__1)
+	);
+Decorate(ExpressionSwitch ::= "switch" Expression__0 CaseSwitch+ "default" Expression__1) -> SemanticExpressionOperation
+	:= (SemanticExpressionOperation[operator=SWITCH]
+		Decorate(Expression__0)
+		...ParseList(CaseSwitch, SemanticCaseSwitch)
+		Decorate(Expression__1)
+	);
```

## TypeOf
```
Type TypeOf(SemanticExpressionOperation[operator=SWITCH] expr) :=
	1. *If* `expr.children.count` is 1:
		1. *Let* `i` be 0.
	2. *Else:*
		1. *Assert:* `expr.children.count` is greater than or equal to 2.
		2. *Let* `i` be 1.
	3. *Let* `union` be an empty sequence of `Type`s.
	4. *While* `i` is less than `expr.children.count - 1`:
		1. *Let* `type_consequent` be the result of performing `TypeOf(expr.children[i].children.-1)`.
		2. Push `type_consequent` to `union`.
		3. Increment `i`.
	5. *Let* `type_default` be the result of performing `TypeOf(expr.children.-1)`.
	6. Push `type_default` to `union`.
	7. *Return:* `TypeUnion(union)`.
;
```

## Interpret
```
Object! Interpret(SemanticExpressionOperation[operator=SWITCH] expr) :=
	1. *If* `expr.children.count` is 1:
		1. *Note:* There are no cases, so the switch will not be assessed.
		2. *Let* `i` be 0.
	2. *Else:*
		1. *Assert:* `expr.children.count` is greater than or equal to 2.
		2. *If* `expr.children.0` is a `SemanticCaseSwitch`:
			1. *Let* `assess_switch` be `true`.
		3. *Else:*
			1. *Assert:* `expr.children.0` is a `SemanticExpression`.
			2. *If* `expr.children.count` is greater than 2:
				1. *Let* `assess_switch` be *Unwrap:* `Interpret(expr.children.0)`.
		4. *Let* `i` be 1.
	3. *Let* `indeterminate` be *none*.
	4. *While* `i` is less than `expr.children.count - 1`:
		1. *Assert:* `assess_switch` is set.
		2. *Let* `case` be `expr.children[i]`.
		3. *Assert:* `case.children.count` is greater than or equal to 2.
		4. *Let* `j` be 0.
		5. *While* `j` is less than `case.children.count - 1`:
			1. *Let* `assess_option` be `Interpret(case.children[j])`.
			2. *If* `assess_option` is an abrupt completion:
				1. *Set* `indeterminate` to `assess_option`.
				2. Increment `j`.
				3. *Continue.*
			3. *If* *UnwrapAffirm:* `Equal(assess_switch, assess_option.value)`:
				1. *Return:* `Interpret(case.children.-1)`.
			4. Increment `j`.
		6. Increment `i`.
	5. *If* `indeterminate` is not *none*:
		1. *Assert:* `indeterminate` is an abrupt completion.
		2. *Note:* At this point the algorithm was unable to assess at least one option for at least one case,
			and has not yet found a match,
			so it is possible, but not determinable, whether one of the cases match the switch.
		3. *Throw:* `indeterminate`.
	6. *Return:* `Interpret(expr.children.-1)`.
;
```
## Build
```
Sequence<Instruction> Build(SemanticExpressionOperation[operator=SWITCH] expr) :=
	1. *Let* `sequence` be an empty sequence of `Instruction`s.
	2. *Let* `end` be an empty sequence of `Instruction`s.
	3. *If* `expr.children.count` is 1:
		1. *Note:* There are no cases, so the switch will not be evaluated.
		2. *Let* `instrs_switch` be ["NOOP"].
		3. *Let* `i` be 0.
	4. *Else:*
		1. *Assert:* `expr.children.count` is greater than or equal to 2.
		2. *If* `expr.children.0` is a `SemanticCase`:
			1. *Let* `instrs_switch` be `1`. // the stack value `1` represents the Counterpoint value `true`
		3. *Else:*
			1. *Assert:* `expr.children.0` is a `SemanticExpression`.
			2. *If* `expr.children.count` is greater than 2:
				1. *Let* `instrs_switch` be *UnwrapAffirm:* `Build(expr.children.0)`.
			3. *Else:*:
				1. *Note:* There are no cases, so the switch will not be evaluated.
				2. *Let* `instrs_switch` be ["NOOP"].
		4. *Let* `i` be 1.
	5. Push ...[
		...`instrs_switch`,
		"SET the local variable `operand0`.",
	] to `sequence`.
	6. *While* `i` is less than `expr.children.count - 1`:
		1. *Let* `case` be `expr.children[i]`.
		2. *Assert:* `case.children.count` is greater than or equal to 2.
		3. *Let* `instrs_option` be *UnwrapAffirm:* `Build(case.children[0])`.
		4. Push ...[
			"GET the local variable `operand0`.",
			`...instrs_option`,
			"EQ",
		] to `sequence`.
		5. *Let* `j` be 1.
		6. *While* `j` is less than `case.children.count - 1`:
			1. *Set* `instrs_option` to *UnwrapAffirm:* `Build(case.children[j])`.
			2. Push ...[
				"GET the local variable `operand0`.",
				...`instrs_option`,
				"EQ",
				"OR",
			] to `sequence`.
			3. Increment `j`.
		7. *Let* `instrs_consequent` be *UnwrapAffirm:* `Build(case.children.-1)`.
		8. Push ...[
			"IF",
			...`instrs_consequent`,
			"ELSE"
		] to `sequence`.
		9. Push "END" to `end`.
		10. Increment `i`.
	7. *Let* `instrs_default` be UnwrapAffirm:* `Build(expr.children.-1)`.
	8. Push ...[
		...`instrs_default`,
		...`end`,
	] to `sequence`.
	9. *Return:* `sequence`.
;
```
