A synchronous block expression, or “sync block” for short, is a block of statements that produces an expression.
```cp
let sum: int = sync {
	print.("I will evaluate to 42.");
	42;
} + 1;
sum; %== 43
```
The block expression above is a synchronous block (asynchronous blocks are coming in a future release) that contains two statements: a function call and an expression-statement. The last statement in a sync block determines the value of the block, so it has a special name: the **determinant**. In this case the determinant is `42`, making the variable `sum` equal to `43`. (Think of the block as “returning” `42`, but this is not the same as containing an actual `return` statement.)

## Block Values and Types
The determinant of a sync block is syntactic. Whatever the last statement happens to be, will be the value of the block. Most of the time that last statment will be an expression-statement, producing the value of the block. However, if the last statement is something else, the block has no value!
```cp
let blex: int = sync {
	for i from 0 to 10 do {
		i * 2;
	};
}; %> TypeError
```
Much like a void function call, this code is invalid because we’re trying to assign a non-value.

Relatedly, a sync block might never finish evaluation!
```cp
let blex: int = sync {
	for i from 0 to 5 by -1 do { % infinite loop!
		i * 2;
	};
	42;
};
```
In both examples, control flow is able to reach the end of the sync block and type information may be gathered. The difference is that in the latter example, a type is able to be determined, and thus the assignment is allowed at compile-time. Static analysis considers the type of the block `int` because its determinant is `42;` and the compiler might be unaware that the loop is infinite. Sync blocks that contain infinite loops or that throw errors or otherwise end abruptly are akin to functions whose return type is `never`.

It might be desirable to return early, like we can in functions. In this situation we may assign the value to a variable and then use the variable as the determinant.
```cp
let blex: int = sync {
	let ret: int = 0;
	% figure out some_condition
	if some_condition then {
		% do some logic…
		set ret = 42;
	};
	ret;
};
```

A sync block *must not* contain a void function call as its determinant — `void` is not a valid expression type. There is one exception: when the block is returned directly from another void function. See #46 for details.
```cp
let n: unknown = sync {
	print.("void"); %> Error
};

function return_void(): void {
	print.("tail optimization FTW!");
	return sync {
		let v: str = "void";
		print.(v); % ok
	};
}

my_list.forEach.((item) {
	return sync {
		print.("about to print item.");
		print.(item); % ok
	};
});
```
Remember that a lambda with a single explicit return statemnt may be converted into an implicit return (with a fat arrow) …
```cp
my_list.forEach.((item) => sync {
	print.("about to print item.");
	print.(item);
});
```
… and this might look familiar:
```cp
my_list.forEach.((item) {
	print.("about to print item.");
	return print.(item);
});
```
Theoretically, every function with an explicit block body may be translated into a function with an implicit return of a sync block, and vice versa.

## Comparison to IIFEs
Sync blocks are more flexible than IIFEs (“immediately-invoked function expressions”) in that they don’t need parameters or captures and may contain **abrupt completions**. Sync blocks may contain `return`, `break`, `continue`, and `throw` statements (depending on lexical context); these are all known as “abrupt” completions, because they abruptly transfer control out of the block without finishing the evaluation of it. In sync blocks, these statements are scoped to their *containing lexical environment*, rather than their own.

For example, much like a regular block, a block expression may contain a `break` statment if inside a loop.
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
The `break;` statement is scoped to the `while` block, not the `if` block. Similarly, if we have a sync block inside a `while` loop, a `break;` statement applies to the loop.
```cp
function f(i: int): void {
	while true do {
		set i += 1;
		let message: bool = mod.(i, 3) == 0 && sync {
			break;
		};
	};
	print.(i);
}
```
The `break;` statment would be invalid if the sync block were not inside a loop. Note that the sync block never finishes evaluating and so has no determinant; thus the variable `message` never gets initialized. However, the loop does break and `i` gets printed.

Because the end of the sync block is unreachable via control flow analysis, the block’s type is `never` and is still assignable to the variable. If the block didn’t complete abruptly and didn’t have the correct type of determinant then the compiler would raise a TypeError, as shown at the beginning of this section.

Abrupt completions also apply to `continue` and `return` statements. When a sync block contains a `return` statement, it tells its containing function to return.
```cp
function f(): int {
	let message1 = "one";
	let message2: str = sync {
		"two";
		return 2;
	};
	let message3 = "three";
}
```
In this code, the sync block returns `2` from the function abruptly. At runtime, `message1` gets set, but then the function returns, and the last two variables are never get initialized. Notice the sync block *didn’t* produce `2` as its determinant and assign it to `message2`, completing execution and moving on to the third variable.

The code passes static analysis because, as above, the sync block has an abrupt completion and has type `never`, which is assignable to `str`. (In fact we didn’t even need the `"two";` statement in it.) However, the return value is still analyzed. If we attempted to return a boolean for example, we would get a TypeError.
