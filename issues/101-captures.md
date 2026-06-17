Functions can be made **stateful** by **capturing** variables from outside their scope.

By default, function scopes do not have access to any variables declared outside. But with a **capture clause**, variables can be cherry-picked and read inside the function body. The capture clause is a bracketed list of previously-declared variables, written before the function parameters (and after the generic parameters). Functions cannot capture from below.
```cpl
val mut my_var:       str       = "hello";
val     my_vars_list: mut [str] = ["world"];

func my_fn(): void {
	my_var;       %> ReferenceError: `my_var` is out of scope. Did you mean to capture it?
	my_vars_list; %> ReferenceError: `my_vars_list` is out of scope. Did you mean to capture it?
	return;
}

func my_fn(): void with (my_var, my_vars_list) {
%                  ^ capture clause
	my_var;       %== "hello"
	my_vars_list; %== ["world"]
	return;
}

func from_below(): str with (my_var_below) => my_var; % ReferenceError: `my_var_below` is used before it is declared.
val my_var_below: str = "hello world";
```

Even if a captured variable is writable (declared with `mut`), it may not be reassigned within the function body by default. (More on that in the next section.)
```cpl
val mut my_var:       str       = "hello";
val     my_vars_list: mut [str] = ["world"];

func my_fn(): void with (my_var, my_vars_list) {
	set my_var       = "ciao!";            %> AssignmentError
	set my_vars_list = ["hello", "world"]; %> AssignmentError
	return;
}
```

Under the hood: When a function captures a variable, a new reference to the variable’s value is copied and bound to the function. If the function is moved around (e.g., returned from another function, or sent as an argument), the new binding moves with it, so it can be read whenever the function is called. If the old reference is destroyed, the copied reference stays with the function for as long as it’s alive.
```cpl
func main(): \() => str {
	val my_var: str = "hello";

	return \(): str with (my_var) => my_var; % `my_var` is copied and bound to the lambda
	% leaving `main` scope, reference `my_var` is destroyed
}
val my_fn: \() => str = main();
my_fn(); %== "hello"
```

Because a new reference is created, if the original reference is ever reassigned after it’s captured, the reassignment *will not* be observed by the function call. However, all mutations, whether on a read-only or writable variable, *will* be observed.
```cpl
val mut my_var:       str       = "";
val     my_vars_list: mut [str] = ["world"];

set my_var = "hello"; % reassigning before capturing will affect the capture

func my_fn(): [str] with (my_var, my_vars_list) { % copies and binds new references
	my_var;       %== "hello"
	my_vars_list; %== ["world"]
	return [my_var, my_vars_list.[0]];
}

set my_var           = "ciao!";  % reassigning an already-captured variable does not affect the function binding,
set my_vars_list.[0] = "mondo!"; % but mutating one does!

my_fn(); %== ["hello", "mondo!"]
```

Unbound variables in default parameter values must be captured.
```cpl
val my_var: str = "hello";

func make_question(s?: str = my_var): str with (my_var) => """{{ s }}?""";

make_question(); %== "hello?"

set my_var = "ciao"; % has no effect, since `my_var` was already copied

make_question(); %== "hello?"
```
Note that default parameters are still re-evaluated on each function call (see #55). As a minimal example:
```cpl
func my_fn(): str {
	print("hello")
	return "world";
}

func make_question(s?: str = my_fn()): str => """{{ s }}?""";
%                            ^ named functions cannot be captured since they aren’t “real” bindings

make_question();     % prints `"hello"`, returns `"world?"`
make_question("hi"); % prints nothing,   returns `"hi?"`
make_question();     % prints `"hello"`, returns `"world?"`
```
In the example above, capturing `my_fn` was not needed, and not even possible, since it is a named function, not a variable. In Counterpoint, named functions (function declarations) and named classes are static, so they don’t have pointers. They’re not true environment bindings. However, as discussed in #46, they cannot be passed around like anonymous functions (function expressions); they must be called when referenced.

## Shared Captures
Variable references can be **shared** with function scopes via the `ref` modifier. These captures are *not* copies of variable references, but are the actual references themselves, accessible within the function. Shared captures are essentially aliases. Reassignments to them in either scope affect the other.
```cpl
val mut my_var:       str       = "hello";
val     my_vars_list: mut [str] = ["world"];

func my_fn1(): str with (ref my_var, my_vars_list) {
%                        ^ shared capture
	set my_var       = "ciao";             % reassigment is allowed, and it affects outer scope
	set my_vars_list = ["hello", "world"]; %> AssignmentError % outer variable is not writable

	return """{{ my_var }}!""";
}

my_fn1(); %== "ciao!"

assert my_var == "ciao";

func my_fn2(): void with (ref my_var)
	=> print(my_var);

my_fn2(); % prints "ciao"

set my_var = "hi"; % affects `my_fn1` and `my_fn2` scope

my_fn2(); % prints "hi"
```

Only declared functions may use shared captures. It’s a syntax error to use shared captures with function expressions.
```cpl
val my_var: str = "hello";
val my_lambda: \() => str = \(): str with (ref my_var) => my_var; %> ParseError
val my_lambda: \() => str = \(): str with (my_var)     => my_var; % ok
```

The reasoning is this: Because a shared capture does not create a copy of the variable reference, it does not increment or decrement the VM’s reference counter. Therefore any function that is defined with shared captures must not leave its defining scope, otherwise it would be memory-unsafe. Declared functions are already unmovable (in fact they’re not even first-class values), so the use of shared-captures is restricted to these functions only.

Consider the possibility (for discussion purposes only):
```cpl
func returns_lambda(): \() => str {
	val my_var: str = "hello";

	% If this were allowed:
	return \(): str with (ref my_var) => my_var;
	% leaving `returns_lambda` scope, reference `my_var` is destroyed
}
returns_lambda()(); % crash! trying to access destroyed `my_var` reference

val my_vars_list: mut [str] = ["world"];
func takes_lambda(lambda: \() => void): void
	=> lambda();
% If this were allowed:
takes_lambda(\() with (ref my_vars_list) { my_vars_list; });
% lambda could be called after `my_vars_list` is destroyed
```
In the examples above, if we were allowed to return the function from `returns_lambda` then it would not have any reference to `my_var` after it is destroyed. Similarly, sending a lambda into `takes_lambda` with a shared capture is also problematic, because the lambda might outlive this scope (if, for example, `takes_lambda` were sent to another scope). Similar problems would occur if share-capturing functions were allowed to be stored in longer-living objects (such as Lists).

```ebnf
CaptureSpecifier<Ref> ::= "with" "(" ","? (<Ref+>"ref"? IDENTIFIER)# ","? ")";

DeclaredFunction<Heritage, Instance, Method> ::= GenericSpecifier<?Instance>? DeclaredFnParams<?Heritage><?Instance><?Method> ... CaptureSpecifier<+Ref>?;
ExpressionFunction<Typed, Instance, Method>  ::= GenericSpecifier<?Instance>? ExpressionFnParams<?Typed><?Instance><?Method>  ... CaptureSpecifier<-Ref>?;

Class<Abstract, Final, Data, Declared, Instance> ::=
	& "class"
	...
	& <Declared->CaptureSpecifier<-Ref>?
	& <Declared+>CaptureSpecifier<+Ref>?
	...
	& "{"
	...
	& "}"
;
```

It’s pretty common for lambdas to need shared captures. Take this simple example of incrementing a global counter in a `.forEach` callback:
```cpl
val mut counter: int = 0;
my_list.forEach(\() with (ref counter) { %> SyntaxError
	set counter += 1;
});
```
`ref`-captures in lambdas are not only invalid, they’re syntactically forbidden by the Counterpoint grammar.

The most sensible workaround would be to write a declared function with the shared capture that “does the work”, and then call that fuction in the callback. Remember, declared functions aren’t capturable since they’re not true bindings, so we can reference it in other functions.
```cpl
val mut counter: int = 0;
func increment(): void with (ref counter) {
	set counter += 1;
}
my_list.forEach(\() {
	increment();
	return;
});
```
This pattern plays by the rules, and is safe when it comes to reference counting. The declared function stays in the scope of its shared capture, and then is invoked in the callback which is passed to a method.

If you don’t like adding a new function to your code, another potential workaround would be to use a mutable copy-captured object to track shared state. This example copy-captures an object that contains the counter.
```cpl
val wrapped_counter: mut Dict.<int> = [value= 0];
my_list.forEach(\() with (wrapped_counter) {
	wrapped_counter.set(@value, wrapped_counter.get(@value) + 1);
});

% or use an interface type:
val wrapped_counter: mut interface { value: int; } = [value= 0];
my_list.forEach(\() with (wrapped_counter) {
	set wrapped_counter.value += 1;
});
```

## Closures
**Closures** are functions that are executed in a different scope from where they were defined.
These could be functions returned by a higher-order function, or functions that are themselves captured
into another other control structure or function.

One kind of closure is a function that captures a variable and then is returned by a higher-order function.
When the closure is called in the outer scope, the captured variable could become visible to that scope.
```cpl
func return_closure(): \(b: int) => int {
	val a: int = 5;
	return \($b) with (a) => a + b;
}
```
The returned function expression captures `a` from its containing scope.
When we call the closure, we have (indirect) access to `a`.
```cpl
val closure: \(b: int) => int = return_closure();
assert closure(b= 8) == 13;
```
We can infer the value of `a` by subtracting 8 from 13.

The code above showed an example of a closure being called “outside” the scope in which it was defined.
Another kind of closure is a function that is called in an “inner” scope.
```cpl
val b: int = 3;
func plus5(): int with (b) => b + 5;
```
The closure `plus5` is defined in the outermost scope and captures `b` from that scope.
When we call it within a new scope below, we have indirect access to `b`.
```cpl
func capture_closure(): void {
	assert plus5() == 8;
}
```
(Note: `capture_closure` cannot capture `plus5` since `plus5` is a declared function and declared functions are not capturable.)
Inside this scope, we can infer the value of `b` without explicitly capturing it, by subtracting 5 from 8.
