Functions can be made **stateful** by **capturing** variables from outside their scope.

By default, function scopes do not have access to any variables declared outside. But with a **capture clause**, variables can be cherry-picked and read inside the function body. The capture clause is a bracketed list of previously-declared variables, written before the function parameters. Functions cannot capture from below.
```cp
let var my_var:       str       = "hello";
let     my_vars_list: mut str[] = ["world"];

function my_fn(): void {
	my_var;       % ReferenceError: `my_var` is out of scope. Did you mean to capture it?
	my_vars_list; % ReferenceError: `my_vars_list` is out of scope. Did you mean to capture it?
}

function my_fn[my_var, my_vars_list](): void {
%             ^ capture clause
	my_var;       %== "hello"
	my_vars_list; %== ["world"]
}

function from_below[my_var_below](): str => my_var; % ReferenceError: `my_var` is used before it is declared.
let my_var_below: str = "hello world";
```

Even if a captured variable is unfixed (declared with `var`), it may not be reassigned within the function body by default. (More on that in the next section.)
```cp
let var my_var:       str       = "hello";
let     my_vars_list: mut str[] = ["world"];

function my_fn[my_var, my_vars_list](): void {
	set my_var       = "ciao!";            %> AssignmentError
	set my_vars_list = ["hello", "world"]; %> AssignmentError
}
```

Under the hood: When a function captures a variable, a new reference to the variable’s value is copied and bound to the function. If the function is moved around (e.g., returned from another function, or sent as an argument), the new binding moves with it, so it can be read whenever the function is called. If the old reference is destroyed, the copied reference stays with the function for as long as it’s alive.
```cp
function main(): (() => str) {
	let my_var: str = "hello";

	return [my_var](): str => my_var; % `my_var` is copied and bound to the lambda
	% leaving `main` scope, reference `my_var` is destroyed
}
let my_fn: () => str = main.();
my_fn.(); %== "hello"
```

Because a new reference is created, if the original reference is ever reassigned after it’s captured, the reassignment *will not* be observed by the function call. However, all mutations, whether on a fixed or unfixed variable, *will* be observed.
```cp
let var my_var:       str       = "";
let     my_vars_list: mut str[] = ["world"];

set my_var = "hello"; % reassigning before capturing will affect the capture

function my_fn[my_var, my_vars_list](): str[2] { % copies and binds new references
	my_var;       %== "hello"
	my_vars_list; %== ["world"]
	return [my_var, my_vars_list.[0]];
}

set my_var           = "ciao!";  % reassigning an already-captured variable does not affect the function binding,
set my_vars_list.[0] = "mundo!"; % but mutating one does!

my_fn.(); %== ["hello", "mundo!"]
```

Unbound variables in default parameter values must be captured.
```cp
let my_var: str = "hello";

function make_question[my_var](s?: str = my_var): str => """{{ s }}?""";

make_question.(); %== "hello?"

set my_var = "ciao"; % has no effect, since `my_var` was already copied

make_question.(); %== "hello?"
```
Note that default parameters are still re-evaluated on each function call (see #55). As a minimal example:
```cp
function my_fn(): str {
	print.("hello")
	return "world";
}

% copies and binds `my_fn` to `make_question`, but does not evaluate `my_fn.()` here
function make_question[my_fn](s?: str = my_fn.()): str => """{{ s }}?""";

make_question.();     % prints `"hello"`, returns `"world?"`
make_question.("hi"); % prints nothing,   returns `"hi?"`
make_question.();     % prints `"hello"`, returns `"world?"`
```

## Shared Captures
Variable references can be **shared** with function scopes via the `ref` modifier. These captures are *not* copies of variable references, but are the actual references themselves, accessible within the function. Shared captures are essentially aliases. Reassignments to them in either scope affect the other.
```cp
let var my_var:       str       = "hello";
let     my_vars_list: mut str[] = ["world"];

function my_fn1[ref my_var, my_vars_list](): str {
%               ^ shared capture
	set my_var       = "ciao";             % reassigment is allowed, and it affects outer scope
	set my_vars_list = ["hello", "world"]; %> AssignmentError % outer variable is not unfixed

	return """{{ my_var }}!""";
}

my_fn1.(); %== "ciao!"

assert my_var == "ciao";

function my_fn2[ref my_var](): void {
	print.(my_var);
}

my_fn2.(); % prints "ciao"

set my_var = "hi"; % affects `my_fn1` and `my_fn2` scope

my_fn2.(); % prints "hi"
```

Only declared functions may use shared captures. It’s a syntax error to use shared captures with function expressions (lambdas).
```cp
let my_var: str = "hello";
let my_lambda: () => str = [ref my_var](): str => my_var; %> ParseError
let my_lambda: () => str = [my_var]():     str => my_var; % ok
```

The reasoning is this: Because a shared capture does not create a copy of the variable reference, it does not increment or decrement the VM’s reference counter. Therefore any function that is defined with shared captures must not leave its defining scope, otherwise it would be memory-unsafe. Declared functions are already unmovable (in fact they’re not even first-class values), so the use of shared-captures is restricted to these functions only.

Consider the possibility (for discussion purposes only):
```cp
function returns_lambda(): (() => str) {
	let my_var: str = "hello";

	% If this were allowed:
	return [ref my_var](): str => my_var;
	% leaving `returns_lambda` scope, reference `my_var` is destroyed
}
returns_lambda.().(); % crash! trying to access destroyed `my_var` reference

let my_vars_list: mut str[] = ["world"];
function takes_lambda(lambda: () => void): void {
	lambda.();
}
% If this were allowed:
takes_lambda.([ref my_vars_list]() { my_vars_list; });
% lambda could be called after `my_vars_list` is destroyed
```
In the examples above, if we were allowed to return the function from `returns_lambda` then it would not have any reference to `my_var` after it is destroyed. Similarly, sending a lambda into `takes_lambda` with a shared capture is also problematic, because the lambda might outlive this scope (if, for example, `takes_lambda` were sent to another scope). Similar problems would occur if share-capturing functions were allowed to be stored in longer-living objects (such as Lists).

```ebnf
CaptureSpecifier<Ref> ::= "[" ","? (<Ref+>"ref"? IDENTIFIER)# ","? "]";

DeclaredFunction<Heritage, Instance, Method> ::= GenericSpecifier<?Instance>? CaptureSpecifier<+Ref>? DeclaredFnParams<?Heritage><?Instance><?Method> ...;
ExpressionFunction<Typed, Instance, Method>  ::= GenericSpecifier<?Instance>? CaptureSpecifier<-Ref>? ExpressionFnParams<?Typed><?Instance><?Method>  ...;

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

The workaround is to use a mutable copy-captured object to track shared state. A typical example would be incrementing a global counter in a `.forEach` callback.
```cp
let var counter: int = 0;
my_list.forEach.([ref counter]() { %> SyntaxError
	set counter += 1;
});

% workaround:
let wrapped_counter: mut Dict.<int> = Dict.<int>([value= 0]); % or use a more structured type, like `mut interface { value: int; }`
my_list.forEach.([wrapped_counter]() {
	set wrapped_counter.[@value] += 1;
});
```
