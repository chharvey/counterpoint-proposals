Generator objects and the `++` (“next”) operator.

# Discussion
A **Generator** is an object that **yields** values one at a time. The motivation for Generators is that they are able to yield items one by one, without having to store them all in a collection (such as a list) at the same time. This can be very benficial in saving runtime memory space. However, some downsides are that Generators cannot backtrack (one cannot view a previous item without saving it), and their access time is *O(n)* rather than *O(1)*.

## Generator Properties
Generators are **iteratables**, a category that also includes tuples and lists. The built-in type for Generators is `Generator<T>`, where the generic parameter `T` represents the type of value yielded.

`Generator` objects have a `count` field, which indicates the number of values yielded so far. It is always a non-negative integer.

`Generator` objects also have a `done` field, which indicates whether the Generator has finished yielding all its specified values. If the Generator is finite, then after it yields all its specified values, it is `done` and will henceforth only yield `null`. Some Generators may be infinite though, which means they’ll never stop yielding and they’ll never be `done`. Note that, depending on how it’s written, a generator could yield the `null` value without being `done`.

The **“next” operator**, `++`, is a postfix unary operator on Generator objects and gives their next yielded value. It has the same precedence as property access and function calls, higher than other unary operators. Thus `?g++` is parsed as `?(g++)`, and `g++.prop.()` is well-formed. This issue introduces a *breaking* change: Since `++` is a new token, expressions such as `++x` will fail parsing and will need to be rewritten as `+(+x)`. (However, given that the unary `+` operator is rarely used twice, the risk is acceptable.)

## Examples
In this example, assume `countdown` is of type `Generator.<int>`.
```cp
let countdown: Generator.<int> = (%% Instantiate a Generator that yields `3`, then `2`, then `1`. %%);
countdown.done;  %== false
countdown.count; %== 0

countdown++;     %== 3
countdown.done;  %== false
countdown.count; %== 1

countdown++;     %== 2
countdown.done;  %== false
countdown.count; %== 2

countdown++;     %== 1
countdown.done;  %== false
countdown.count; %== 3

countdown++;     %== null
countdown.done;  %== true
countdown.count; %== 3

countdown++;     %== null
countdown.done;  %== true
countdown.count; %== 3

countdown++;     %== null
countdown.done;  %== true
countdown.count; %== 3
```
Notice that after a generator is `done`, it continues to yield `null` and doesn’t increment its `count`.

Generators may be iterated over in a `for-of` loop (#28), where the variable represents the yielded items. Notice we don’t need to use the `++` operator, as the `for` loop does this implicitly. The `for` loop stops iteration once the generator is `done`.
```cp
countdown; %: Generator.<int>
let results: mut int[] = List.<int>([]);
let i: int = 0;
for n: int of countdown do {
	set results.[i] = n;
	set i += 1;
};
results; %== [3, 2, 1]
```

## Constructing Generators
The built-in `newGenerator<T>` function constructs `Generator.<T>` objects. (This will be converted to a class in later versions.) `newGenerator` takes an executor function argument, which returns void, and has three parameters: a yielder, a returner, and a counter. In this example, we’re constructing a `Generator.<str>`.
```cp
newGenerator.<str>((yielder, returner, counter) {
	yielder;  %: (str) => void
	returner; %: () => void
	counter;  %: int
});
```
- `yielder`: a void function that takes a value to yield. When called, it increments the counter.
- `returner`: a void function to call when to signify the generator is done.
- `counter`: a non-negative integer that gives read-only access to the generator’s `count` field.

The executor function executes every time the Generator is iterated (every time “next” (`++`) is performed). In this executor function body, we specify which values to yield and when. To construct a Generator that yields `'three'`, then `'two'`, and then `'one'`, we might use the following implementation:
```cp
let countdown: Generator.<str> = newGenerator.<str>((yielder, returner, counter) {
	if counter == 0 then {
		yielder.("three");
	} else if counter == 1 then {
		yielder.("two");
	} else if counter == 2 then {
		yielder.("one");
	} else {
		returner.();
	};
});
```

# Specification

## Lexicon & Syntax
```diff
Punctuator :::=
+	| "++"
;

ExpressionCompound ::=
	| ExpressionUnit
-	| ExpressionCompound (PropertyAccess | FunctionCall)
+	| ExpressionCompound ("++" | PropertyAccess | FunctionCall)
;
```

## Semantics
```diff
+SemanticOperation[operator: NEXT]
+	::= SemanticExpression[type: Generator];
SemanticOperation[operator: NOT | EMP]
	::= SemanticExpression;
SemanticOperation[operator: NEG]
	::= SemanticExpression[type: Number];
```
It is a semantic error if the `NEXT` operator is used on an expression that is not of type `Generator`.

## Decorate
```diff
Decorate(ExpressionCompound ::= ExpressionUnit) -> SemanticExpression
	:= Decorate(ExpressionUnit);
+Decorate(ExpressionCompound ::= ExpressionCompound "++") -> SemanticOperation
+	:= (SemanticOperation[operator=NEXT]
+		Decorate(ExpressionCompound)
+	);
Decorate(ExpressionCompound ::= ExpressionCompound PropertyAccess) -> SemanticAccess
	:= (SemanticAccess[kind=AccessKind(PropertyAccess)]
		Decorate(ExpressionCompound)
		Decorate(PropertyAccess)
	);
Decorate(ExpressionCompound ::= ExpressionCompound FunctionCall) -> SemanticCall
	:= (SemanticCall
		Decorate(ExpressionCompound)
		...(...Decorate(FunctionCall))
	);
```

## TypeOf
```diff
Type! TypeOfUnfolded(SemanticOperation[operator: NEXT] expr) :=
	1. *Assert:* `expr.children.count` is 1.
	2. *Let* `base_type` be *Unwrap:* `TypeOf(expr.children.0)`.
	3. *If* `base_type` is a Generator type:
		1. *Return:* the yield type of `base_type`.
	4. *Else:*
		1. *Throw:* a new TypeErrorInvalidOperation.
;
```

## Core
```cp
type Generator<T> = [
	count:    int,
	done:     bool,
	executor: ((T) => void, () => void, int) => void,
];

function newGenerator<T>(executor: ((T) => void, () => void, int) => void): Generator.<T> => [
	count=    0,
	done=     false,
	executor= executor,
];
```

The following algorithm will be executed at runtime every time the “next” operator `++` is performed on a Generator instance.
```
Object Next(Generator generator) :=
	1. *Let* `v` be `null`.
	2. *Let* `yielder` be a new Closure defined by the following steps:
		1. Given a parameter `value`,
		2. Set `v` to `value`.
		3. Increment `generator.count` by 1.
	3. *Let* `returner` be a new Closure defined by the following steps:
		1. Set `generator.done` to `true`.
	4. *Perform* *Unwrap:* `CallFunction(generator.executor, [yielder, returner, generator.count])`.
	5. *Return:* `v`.
;
```
