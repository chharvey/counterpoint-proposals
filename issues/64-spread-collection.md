Spread syntax is a way of injecting collection entries into other collections.

# Discussion

## Fixed-Size Spread

### Tuple Spread
Tuples may be **spread** into other tuple literals, wherin the items of the first tuple are injected into the second tuple, in order, where the spread syntax is.
```cp
let cde: str[3] = [c, d, e]; % assume these variables are strings
let abcdefg = str[7] = [a, b, #cde, f, g];
abcdefg == [a, b, c, d, e, f, g];
```
The symbol `#` is called the **single-spread** symbol. It’s technically not an operator, since it doesn’t produce an expression, and it doesn’t have the same precedence as unary operators. For example, `[#a || b, c]` is interpreted as `[#(a || b), c]`.

Tuples may also be spread into set literals.
```cp
let cde: [str, str, str] = [c, d, e]; % assume these variables are strings
let def = str{} = {d, e, #cde, f};
def == {d, e, c, f};
```

### Record Spread
Records can be spread into other records, but since records have keys, the rules for record spreading is stricter than for tuple spreading. Record spreading uses the symbol `##`, and duplicate record keys are overwritten in source order. The symbol `##` is called the **double-spread** symbol (again, not an actual operator).
```cp
type Car = [
	make:  str,
	model: str,
	color: str,
	year:  int,
];
let my_car: Car = [
	make=  "Mazda",
	model= "Protegé",
	color= "black",
	year=  2003,
];
let new_car: Car = [
	color= "red",
	##my_car,
	year= 2010,
];
new_car == [
	make=  "Mazda",
	model= "Protegé",
	color= "black",
	year=  2010,
];
```
Notice that `new_car.color == "black"` but `new_car.year == 2010`. This is because the spread `##my_car` overwrote the `color = "red"`, but the new year overwrote the year in `my_car`. Properties later in source order have higher precedence. When multiple records are spread, properties cascde over each other in source order.
```cp
let newer_car: Car = [
	make=  "Honda",
	model= "Civic",
	color= "blue",
	year=  2008,
	##[make= "Mazda", model= "Protegé"], % overwrite make and model
	##[year= 2010],                      % overwrite year
];
newer_car == [
	make=  "Mazda",
	model= "Protegé",
	color= "blue",
	year=  2010,
]
```

## Variable-Size Spread

### List Spread
Lists can be spread into sets only.
```cp
let cde: str[] = List.<str>([c, d, e]); % assume these variables are strings
let def = str{} = {d, e, #cde, f};
def == {d, e, c, f};
```
Since there is not yet a literal construction syntax for lists, lists cannot be spread into lists.
```cp
let cde: str[] = List.<str>([c, d, e]); % assume these variables are strings
let abcdefg = str[] = List.<str>([a, b, #cde, f, g]); %> error (see below): cannot spread list into tuple
```

### Dict Spread
Likewise, we cannot yet spread dicts into dicts.
```cp
let my_car: [:str | int] = Dict.<str | int>([
	make=  "Mazda",
	model= "Protegé",
	color= "black",
	year=  2003,
]);
let new_car: [:str | int] = Dict.<str | int>([
	color= "red",
	##Dict.<unknown>(my_car), %> error (see below): cannot spread dict into record
	year= 2010,
]);
```

### Set Spread
Sets can be spread into other sets.
```cp
let cde: str{} = {c, d, e}; % assume these variables are strings
let def = str{} = {d, e, #cde, f};
def == {d, e, c, f};
```
Sets cannot yet be spread into lists, since there is not yet a list literal syntax.

### Map Spread
Maps can be spread into one another with the **triple-spread** symbol `###`.
```cp
let office: Map.<C, C> = {
	jim    -> pam,
	dwight -> angela,
	andy   -> erin,
};
let parks: Map.<C, C> = {
	leslie -> ben,
	ann    -> chris,
	april  -> andy,
};
[###office, ###parks] == {
	jim    -> pam,
	dwight -> angela,
	andy   -> erin,
	leslie -> ben,
	ann    -> chris,
	april  -> andy,
};
```
Maps overwrite cases in the same way that records overwrite properties.

## Errors
We get syntax errors by using the wrong arity of spread *inside* the wrong syntaxes…
```cp
[1,  ##x]; %> ParseError: double-spread inside tuple literal
[1, ###x]; %> ParseError: triple-spread inside tuple literal
[ ##x, 1]; %> ParseError: expression inside record literal
[###x, 1]; %> ParseError: triple-spread inside tuple/record literal
[a= 1,   #y]; %> ParseError: single-spread inside record literal
[a= 1, ###y]; %> ParseError: triple-spread inside record literal
[  #y, a= 1]; %> ParseError: property inside tuple literal
[###y, a= 1]; %> ParseError: triple-spread inside tuple/record literal
{1,  ##x}; %> ParseError: double-spread inside set literal
{1, ###x}; %> ParseError: triple-spread inside set literal
{ ##x, 1}; %> ParseError: double-spread inside set/map literal
{###x, 1}; %> ParseError: expression inside map literal
{"a" -> 1,  #z}; %> ParseError: single-spread inside map literal
{"a" -> 1, ##z}; %> ParseError: double-spread inside map literal
{ #z, "a" -> 1}; %> ParseError: case inside set literal
{##z, "a" -> 1}; %> ParseError: double-spread inside set/map literal
```
and we get type errors by using the wrong arity of spread *on* the wrong types:
```cp
[1, #[b= 2, c= 3]];         %> TypeError: single-spread on record type
[1, #{"b" -> 2, "c" -> 3}]; %> TypeError: single-spread on map type
[a= 1, ##[2, 3]];               %> TypeError: double-spread on tuple type
[a= 1, ##{2, 3}];               %> TypeError: double-spread on set type
[a= 1, ##{"b" -> 2, "c" -> 3}]; %> TypeError: double-spread on map type
{"a" -> 1, ###[2, 3]};       %> TypeError: triple-spread on tuple type
{"a" -> 1, ###{2, 3},};      %> TypeError: triple-spread on set type
{"a" -> 1, ###[b= 2, c= 3]}; %> TypeError: triple-spread on record type
```

Since the counts of tuples are static (known at compile-time), we get type errors where counts don’t match up.
```cp
let failure: str[5] = [a, #[c, d, e]];             %> TypeError (count 4 not assignable to count 5)
let failure: str[3] = [a, #[c, d, e]];             %> TypeError (count 4 not assignable to count 3)
let failure: str[5] = [a, #List.<str>([c, d, e])]; %> TypeError (count unknown int not assignable to count 5)
let failure: str[3] = [a, #List.<str>([c, d, e])]; %> TypeError (count unknown int not assignable to count 3)
```
Similarly for records, since record keys are static.
```cp
let failure: [a: int, b: bool, c: str] = [a= 42, ##[b= "bool", c= true]];   %> TypeError: true not assignable to str
let failure: [a: int, b: bool, c: str] = [a= 42, ##[b= true, d= "string"]]; %> TypeError: missing property `c`
```

# Specification

## Lexicon
```diff
Punctuator :::=
	// unary
+		| "#" | "##" | "###"
;
```

## TokenWorth
```diff
+TokenWorth(Punctuator :::= "#")   -> RealNumber := \x06;
+TokenWorth(Punctuator :::= "##")  -> RealNumber := \x07;
+TokenWorth(Punctuator :::= "###") -> RealNumber := \x08;
-TokenWorth(Punctuator :::= "!")   -> RealNumber := \x06;
-TokenWorth(Punctuator :::= "?")   -> RealNumber := \x07;
+TokenWorth(Punctuator :::= "!")   -> RealNumber := \x09;
+TokenWorth(Punctuator :::= "?")   -> RealNumber := \x0a;
# and so on…
```

## Syntax
```diff
-TupleLiteral  ::= "[" ( ","?                     Expression#  ","? )? "]";
-RecordLiteral ::= "["   ","?                     Property#    ","?    "]";
-SetLiteral    ::= "{" ( ","?                     Expression#  ","? )? "}";
-MapLiteral    ::= "{"   ","?                     Case#        ","?    "}";
+TupleLiteral  ::= "[" ( ","? ("#"   Expression | Expression)# ","? )? "]";
+RecordLiteral ::= "["   ","? ("##"  Expression | Property)#   ","?    "]";
+SetLiteral    ::= "{" ( ","? ("#"   Expression | Expression)# ","? )? "}";
+MapLiteral    ::= "{"   ","? ("###" Expression | Case)#       ","?    "}";
```

## Semantics
```diff
+SemanticSpread[arity: 1 | 2 | 3]
+	::= Expression;

-SemanticTuple   ::=  SemanticExpression*;
-SemanticRecord  ::=  SemanticProperty+;
-SemanticSet     ::=  SemanticExpression*;
-SemanticMap     ::=  SemanticCase+;
+SemanticTuple   ::= (SemanticExpression | SemanticSpread[arity: 1])*;
+SemanticRecord  ::= (SemanticProperty   | SemanticSpread[arity: 2])+;
+SemanticSet     ::= (SemanticExpression | SemanticSpread[arity: 1])*;
+SemanticMapping ::= (SemanticCase       | SemanticSpread[arity: 3])+;
```

## Decorate
```diff
+	Decorate(TupleLiteral__0__List ::= "#" Expression) -> Vector<SemanticSpread>
+		:= [(SemanticSpread[arity=1] Decorate(Expression))];
	Decorate(TupleLiteral__0__List ::= Expression) -> Vector<SemanticExpression>
		:= [Decorate(Expression)];
+	Decorate(TupleLiteral__0__List ::= TupleLiteral__0__List "," "#" Expression) -> Sequence<SemanticExpression | SemanticSpread>
+		:= [
+			...Decorate(TupleLiteral__0__List),
+			(SemanticSpread[arity=1] Decorate(Expression)),
+		];
-	Decorate(TupleLiteral__0__List ::= TupleLiteral__0__List "," Expression) -> Sequence<SemanticExpression>
+	Decorate(TupleLiteral__0__List ::= TupleLiteral__0__List "," Expression) -> Sequence<SemanticExpression | SemanticSpread>
		:= [
			...Decorate(TupleLiteral__0__List),
			Decorate(Expression),
		];

+	Decorate(RecordLiteral__0__List ::= "##" Expression) -> Vector<SemanticSpread>
+		:= [(SemanticSpread[arity=2] Decorate(Expression))];
	Decorate(RecordLiteral__0__List ::= Property) -> Vector<SemanticProperty>
		:= [Decorate(Property)];
+	Decorate(RecordLiteral__0__List ::= RecordLiteral__0__List "," "##" Expression) -> Sequence<SemanticProperty | SemanticSpread>
+		:= [
+			...Decorate(RecordLiteral__0__List),
+			(SemanticSpread[arity=2] Decorate(Expression)),
+		];
-	Decorate(RecordLiteral__0__List ::= RecordLiteral__0__List "," Property) -> Sequence<SemanticProperty>
+	Decorate(RecordLiteral__0__List ::= RecordLiteral__0__List "," Property) -> Sequence<SemanticProperty | SemanticSpread>
		:= [
			...Decorate(RecordLiteral__0__List),
			Decorate(Property),
		];

+	Decorate(MapLiteral__0__List ::= "###" Expression) -> Vector<SemanticSpread>
+		:= [(SemanticSpread[arity=3] Decorate(Expression))];
	Decorate(MapLiteral__0__List ::= Case) -> Vector<SemanticCase>
		:= [Decorate(Case)];
+	Decorate(MapLiteral__0__List ::= MapLiteral__0__List "," "###" Expression) -> Sequence<SemanticCase | SemanticSpread>
+		:= [
+			...Decorate(MapLiteral__0__List),
+			(SemanticSpread[arity=3] Decorate(Expression)),
+		];
-	Decorate(MapLiteral__0__List ::= MapLiteral__0__List "," Case) -> Sequence<SemanticCase>
+	Decorate(MapLiteral__0__List ::= MapLiteral__0__List "," Case) -> Sequence<SemanticCase | SemanticSpread>
		:= [
			...Decorate(MapLiteral__0__List),
			Decorate(Case),
		];
```

# Workaround
If this feature is not implemented, a sufficient workround is to manually list out the entries.
```cp
let tup: int[3] = [2, 3, 4];
[1, tup.0, tup.1, tup.2, 5]; % [1, #tup, 5]
```
