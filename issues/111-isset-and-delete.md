`isset` and `delete` keywords are used for working with uninitialized variables (#96) and optional parameters (#55).

# The `isset` Operator
`isset` is a new unary operator that returns a boolean indicating whether or not a variable or parameter is currently set.
```cpl
let var greeting?: str;
assert greeting == null;
assert isset greeting == false;

set greeting = "hello";
assert greeting == "hello";
assert isset greeting == true;
```
`isset` is used to determine whether a variable has been *set*, regardless of its value, even if that value is `null`. This is why testing whether the variable is `== null` is not sufficient.
```cpl
let var greeting?: str | null;
assert greeting == null;
assert isset greeting == false;

set greeting = null;
assert greeting == null; % testing `== null` does not tell us whether the variable is set
assert isset greeting == true;
```
This operator is also useful for when we need to see whether a value has been supplied for an optional parameter. Suppose instead of using `isset` we were to check the value of the parameter against `null`:
```cpl
function get<T>(list: [T], index: int, 'default'?: T): Maybe.<T>
	=> if list.hasIndex.(index) then Some.<T>(list.[index]) else
	if 'default' != null then Some.<T>('default') else None.<T>();

assert get<str | null>(["hello"], 1)       == None.<str | null>(); % got `None` as expected
assert get<str | null>(["hello"], 1, null) == None.<str | null>(); % got `None`, but was expecting `Some`
```
With `isset`, we get the expected return value.
```cpl
function get<T>(list: [T], index: int, 'default'?: T): Maybe.<T>
	=> if list.hasIndex.(index) then Some.<T>(list.[index]) else
	if isset 'default' then Some.<T>('default') else None.<T>();

assert get<str | null>(["hello"], 1)       == None.<str | null>();     % got `None` as expected
assert get<str | null>(["hello"], 1, null) == Some.<str | null>(null); % got `Some` as expected
```

`isset` may be used to check whether a property has been set on a tuple, record, or statically-known object.
```cpl
let tup: (int, ?: float | null) = (42,);
assert isset tup.0 == true;
assert isset tup.1 == false;

let rec: (a?: int | null, b: float) = (b= 4.2);
assert isset rec.a == false;
assert isset rec.b == true;

let my_object: mut (interface { field?: str; }) = (class { public field?: str; }).();
assert isset my_object.field == false;
set my_object.field = "hello";
assert isset my_object.field == true;
```

For dynamically-indexed objects like Lists, Dicts, Sets, and Maps, use their respective methods instead.
```diff
-assert isset ["hello"].[1] == false;
+assert ["hello"].has.(1) == false;

-assert isset [a= "hello"].[@a] == true;
+assert [a= "hello"].has.(@a) == true;

-assert isset {"hello"}.["world"] == false;
+assert {"hello"}.has.("world")   == false;

-assert isset {"hello" -> 42}.["hello"] == true;
+assert {"hello" -> 42}.hasKey.("hello") == true;
```

## The `!isset` Operator
`!isset a` is syntax sugar for `!(isset a)`.

## Grammar
The `isset` and `!isset` unary operators are given their own position among the other unary keyword operators. They may only be used on `Assignee` nonterminals.
```diff
KeywordOther :::=
	// operator
		...
		| "mut"
+		| "isset"
+		| "!isset"
		| "as"
		| "as?"
		| "as!"
		| "is"
		| "isnt"
		| "if"
		| "then"
		| "else"
#	...
;

PropertyAccessor<Break>
	::= INTEGER | Word | "[" Expression<+Block><?Break> "]";

ExpressionUnaryKeyword<Block, Break> ::=
+	| ("isset" | "!isset") Assignee<?Break>
	| ExpressionUnarySymbol<?Block><?Break>
	| ("int" | "float") ExpressionUnaryKeyword<?Block><?Break>
;

Assignee<Break> ::=
	| IDENTIFIER
	| ExpressionCompound<+Block><?Break> "." PropertyAccessor<?Break>
;
```
Therefore, syntaxes like `?isset a` and `isset (a || b)` are not well-formed, while `?(isset a)` and `isset a || b` are (the latter of which parses to `(isset a) || b`).

# The `delete` Statement
```cpl
let var greeting?: str;
set greeting = null; %> TypeError: Expression of type `null` is not assignable to type `str`.
```
Because the write type of `greeting` is only `str`, we cannot explicitly set its value to `null`. A new `delete` keyword, deletes the value of an uninitialized variable or optional parameter.
```cpl
let var greeting?: str;
assert isset greeting == false;

set greeting = "hello";
assert isset greeting == true;

delete greeting;
assert isset greeting == false;
assert greeting == null;

let var is_done: bool = true;
delete is_done; %> MutabilityError: Deletion of a required variable `is_done`.

function move(x: float, y?: float): void {
	delete x; %> MutabilityError: Deletion of a required parameter `x`.
	delete y; % ok
}
```

Like `isset`, the `delete` statement can only be used for statically-known bound properties on objects.
For Lists, Dicts, Sets, and Maps, which are dynamic, use their respective methods instead.
```diff
-delete [42, 43].[1];
+[42, 43].delete.(1);

-delete [prop= 42].[@prop];
+[prop= 42].delete.(@prop);

-delete {42}.[42];
+{42}.delete.(42);

-delete {"key" -> 42}.["key"];
+{"key" -> 42}.delete.("key");
```

To delete an optional field from an object, use the `delete` keyword with the statically bound property.
```cpl
let my_object: mut (interface { field?: str; }) = (class { public field?: str; }).();
set my_object.field = "hello";
assert my_object.field == "hello";
delete my_object.field;
assert my_object.field == null;
```
This is not quite the same as required fields, which always have values (even if nullish).
```cpl
let my_object: mut (interface { field: str | null; }) = (class { public field: str | null = null; }).();
set my_object.field = "hello";
assert my_object.field == "hello";
set my_object.field = null;
assert my_object.field == null;
delete my_object.field; %> MutabilityError: Deletion of a required field `field`.
```

It’s a semantic error if the bound object is not a mutable type or if the binding property is not optional.
```cpl
delete ("a tuple", "of strings").1;       %> MutabilityError: Mutation of an object of immutable type `(str, str)`.
delete (a_record= "of strings").a_record; %> MutabilityError: Mutation of an object of immutable type `(a_record: str)`.

interface T {
	new ();
	required: str;
	optional?: str;
}

let my_immutable_object: T     = T.();
let my_mutable_object:   mut T = T.();

delete my_immutable_object.optional; %> MutabilityError: Mutation of an object of immutable type `T`.
delete my_mutable_object.opt;        %> TypeError: Property `opt` does not exist on type `mut T`.
delete my_mutable_object.required;   %> MutabilityError: Deletion of a required field `required`.
set my_mutable_object.required = ""; % ok to reassign
```

`delete` is its own statement. Like `isset`, it can only be used on an Assignee nonterminal.
```diff
Assignee<Break> ::=
	| IDENTIFIER
	| ExpressionCompound<+Block><?Break> "." PropertyAccessor<?Break>
;

 DeclarationClaim        <Break> ::= "claim"  Assignee<?Break> ":" Type                       ";";
 DeclarationReassignment <Break> ::= "set"    Assignee<?Break> "=" Expression<+Block><?Break> ";";
+DeclarationDelete       <Break> ::= "delete" Assignee<?Break>                                ";";

Declaration<Break> ::=
	| DeclarationType
	| DeclarationVariable     <?Break>
	| DeclarationClaim        <?Break>
	| DeclarationReassignment <?Break>
+	| DeclarationDelete       <?Break>
;
```
