A block expression is a block of statements that produces an expression.
```cp
let sum: int = {
	print.("I will evaluate to 42.");
	42;
};
sum; %== 42
```
The block expression above contains two statements: a function call and an expression-statement. The last statement in a block expression determines its value, so it has a special name: the **determinant**. In this case the determinant is `42`, which is assigned to the variable `sum`. (Think of the block as “returning” `42`, but this is not the same as containing an actual `return` statement.)

## Operator Precedence
Block expressions are weaker than all other operators. It’s a syntax error to use a block expression as an operand without wrapping it in parentheses. This is similar to the precedence of function expressions (lambdas).
```cp
let greeting: int = { 10; }   + 20 + { 30; };   %> SyntaxError
let greeting: int = ({ 10; }) + 20 + ({ 30; }); % fixed

let callable: () => int = () => 10   || () => 20;   %> SyntaxError
let callable: () => int = (() => 10) || (() => 20); % fixed
```
This avoids syntax ambiguities with conditional statements.
```cp
if condition then { consequent; } else { alternative; };     % a conditional statement
if condition then ({ consequent; }) else ({ alternative; }); % a ternary expression

let result = if a then { b; } else { c; };     %> SyntaxError
let result = if a then ({ b; }) else ({ c; }); % fixed
```

Also note that a block expression is syntactically allowed as the condition, but is not required to be parenthesized.
```cp
if { condition; } then { consequent; } else { alternative; }; % a conditional statement
if { condition; } then consequent else alternative;           % a ternary expression
```

Block expressions must be parenthesized when implicitly returned from a lambda or method.
```cp
my_list.forEach.((item) {
	print.("about to print item.");
	return print.(item);
});

% equivalent to:
my_list.forEach.((item) => ({
	print.("about to print item.");
	print.(item);
}));

% SyntaxError:
my_list.forEach.((item) => {
	print.("about to print item.");
	print.(item);
});
```

## Block Values and Types
The determinant of a block is syntactic. Whatever the last statement happens to be, will be the value of the block. Most of the time that last statement will be an expression-statement, producing the value of the block. However, if the last statement is something else, the block has no value!
```cp
let blex: int = {
	for i from 0 to 10 do {
		i * 2;
	};
}; %> TypeError
```
Much like a void function call, this code is invalid because we’re trying to assign a non-value.

Relatedly, a block expression might never finish evaluation!
```cp
let blex: int = {
	for i from 0 to 5 by -1 do { % infinite loop!
		i * 2;
	};
	42;
};
```
In both examples, control flow is able to reach the end of the block and type information may be gathered. The difference is that in the latter example, a type is able to be determined, and thus the assignment is allowed at compile-time. Static analysis considers the type of the block `int` because its determinant is `42;` and the compiler might be unaware that the loop is infinite. Block expressions that contain infinite loops or that throw errors or otherwise end abruptly are akin to functions whose return type is `never`.

It might be desirable to return early, like we can in functions. In this situation we may assign the value to a variable and then use the variable as the determinant.
```cp
let blex: int = {
	let var ret: int = 0;
	% figure out some_condition
	if some_condition then {
		% do some logic…
		set ret = 42;
	};
	ret;
};
```

A block *must not* contain a void function call as its determinant — `void` is not a valid expression type. There is one exception: when the block is returned directly from another void function. See #46 for details.
```cp
let n: unknown = {
	print.("void"); %> Error
};

function return_void(): void {
	print.("tail optimization FTW!");
	return {
		let v: str = "void";
		print.(v); % ok
	};
}

my_list.forEach.((item) {
	return {
		print.("about to print item.");
		print.(item); % ok
	};
});
```

## Comparison to IIFEs
Block expressions are more flexible than IIFEs (“immediately-invoked function expressions”) in that they don’t need parameters or captures and may contain **abrupt completions**. Block expressions may contain `return`, `break`, `continue`, and `throw` statements (depending on lexical context); these are all known as “abrupt” completions, because they abruptly transfer control out of the block without finishing the evaluation of it. In block expressions, these statements are scoped to their *containing lexical environment*, rather than their own.

For example, much like a regular block, a block expression may contain a `break` statement if inside a loop. First consider this `if` statement, which contains a `break;`.
```cp
function f(i: int): void {
	while true do {
		set i += 1;
		if mod.(i, 3) == 0 then {
			break;
		};
	};
	print.(i);
}
```
The `break;` statement is scoped to the `while` block, not the `then` block. Similarly, if we have a block *expression* inside a `while` loop, a `break;` statement applies to the loop.
```cp
function f(i: int): void {
	while true do {
		set i += 1;
		let message: bool = mod.(i, 3) == 0 && ({
			break;
		});
	};
	print.(i);
}
```
The `break;` statement would be invalid if the block expression were not inside a loop. Note that the block expression never finishes evaluating and so has no determinant; thus the variable `message` never gets initialized. However, the loop does break and `i` gets printed.

Because the end of the block expression is unreachable via control flow analysis, the block’s type is `never` and is still assignable to the variable. If the block didn’t complete abruptly and didn’t have the correct type of determinant then the compiler would raise a TypeError, as shown at the beginning of this section.

Abrupt completions also apply to `continue` and `return` statements. When a block expression contains a `return` statement, it tells its containing function to return.
```cp
function f(): int {
	let message1 = "one";
	let message2: str = {
		"two";
		return 2;
	};
	let message3 = "three";
}
```
In this code, the block expression returns `2` from the function abruptly. At runtime, `message1` gets set, but then the function returns, and the last two variables are never initialized. Notice the block expression *didn’t* produce `2` as its determinant and assign it to `message2`, completing execution and moving on to the third variable.

The code passes static analysis because, as above, the block expression has an abrupt completion and has type `never`, which is assignable to `str`. (In fact we didn’t even need the `"two";` statement in it.) However, the return value is still analyzed. If we attempted to return a boolean for example, we would get a TypeError because it’s not assignable to the return signature of `f`.
