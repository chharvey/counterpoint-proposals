Symbols are global primitive values with opaque representations and without intrinsic semantics.

# Discussion
Symbols are values whose implementations are hidden. They are only referenced by their name.
```cp
.WATER; % a symbol
.FIRE;  % another symbol
```
Symbol names are programmer-defined identifiers, and always follow a dot. They’re conventionally written in MACRO_CASE, but sometimes also in snake_case and in camelCase.

The values of symbols are the only of their kind, and their implementations are unexposed. Symbols can be thought of as integers or strings, but can’t be operated on as such and do not take up as much space. For example, one cannot add symbols and cannot convert them to uppercase. Symbols are not assignable to the `int` or `str` types, and vice versa. Symbols are “value types”, that is, compared and copied by value rather than by reference. Like all values, symbols are assignable to type `unknown`.
```cp
let greet: unknown = .hello_world;

.hello + .world;             %> TypeError
.hello_world.toUppercase.(); %> TypeError
```

Symbol names may be any keyword or identifier, starting with a letter or underscore, and subsequently containing letters, underscores, and/or digits. Even the single underscore `_` is a valid sybmol name. Symbol names may also be Unicode names, but this is not recommended as it hinders readability. As with Unicode identifiers, there are no escapes. When stringified, the the resulting string is the symbol’s name, unescaped.
```cp
let greet: unknown = .'¡héllö wòrld!';

.'\u{24}3.99' != .'$3.99';

let greeting: str = """{{ greet }}""";
greeting; %== "¡héllö wòrld!"

let price: str = """{{ .'\u{24}3.99' }}""";
price; %== """'\u{24}3.99'"""
% notice the string include the symbol’s single-quotes and backslash character
```

Other than `unknown`, symbols are assignable to their unit types. (Unit types are types containing only one value — also referred to as “constant types”.)
```cp
let earth: .EARTH = .EARTH;
```

Symbol unit types can be unioned to form an enumeration of values.
```cp
type Element = .WATER | .EARTH | .FIRE | .AIR;
let var el: Element = .FIRE;
set el = .AIR;
set el = .AETHER; %> TypeError
```
All symbol unit types are disjoint, so their intersection is always `never`.
```cp
type Impossible = .WATER & .EARTH; % type `never`
```

Counterpoint has a designated `symbol` type, which is conceptually the infinite union of all potential symbol values. Any symbol value is assignable to the `symbol` type, and *only* symbol values are assignable to this type.
```cp
let var el: symbol = .FIRE;
set el = .AIR;
set el = .AETHER;
set el = 42;         %> TypeError
set el = "a string"; %> TypeError
```

Symbol values are always truthy and always nonempty.
```cp
!(.WATER) == false; % `!.WATER` is a parse error
?(.FIRE)  == false; % `?.FIRE`  is a parse error

.EARTH || .AIR == .EARTH;
.EARTH && .AIR == .AIR;
```
(Note: The symbols above must be wrapped in parentheses, because the tokens `!.` and `?.` are accessor operators. Or we could use whitespace to separate the operator from the symbol.)
```cp
! .WATER == false; % also acceptable, but maybe less readable
? .FIRE  == false; % also acceptable, but maybe less readable
```

Symbols are equal if and only if their names are exactly the same. As “value types”, symbols are identical if and only if they are equal.
```cp
.LIGHT   !=  .DARK;
.NEUTRAL === .NEUTRAL;
.NEUTRAL !=  .neutral;
```

# Specification

## Lexicon
```diff
Keyword :::=
	// literal
		| "void"
		| "null"
		| "bool"
		| "false"
		| "true"
+		| "symbol"
		| "int"
		| "float"
		| "str"
	// operator
		| "mut"
		| "is"
		| "isnt"
		| "if"
		| "then"
		| "else"
	// storage
		| "let"
		| "type"
	// modifier
		| "var"
;
```

## Syntax
```diff
Word ::=
	| KEYWORD
	| IDENTIFIER
;

+Symbol
+	::= "." Word;

PrimitiveLiteral ::=
	| "null"
	| "false"
	| "true"
	| INTEGER
	| FLOAT
	| STRING
+	| Symbol
;

TypeKeyword ::=
	| "void"
	| "bool"
+	| "symbol"
	| "int"
	| "float"
	| "str"
;
```

## Decorate
```diff
+Decorate(Symbol ::= "." Word) -> SemanticConstant
+	:= (SemanticConstant[value=new Symbol(Decorate(Word).id)]);

Decorate(PrimitiveLiteral ::= "null") -> SemanticConstant
	:= (SemanticConstant[value=null]);
Decorate(PrimitiveLiteral ::= "false") -> SemanticConstant
	:= (SemanticConstant[value=false]);
Decorate(PrimitiveLiteral ::= "true") -> SemanticConstant
	:= (SemanticConstant[value=true]);
Decorate(PrimitiveLiteral ::= INTEGER) -> SemanticConstant
	:= (SemanticConstant[value=new Integer(TokenWorth(INTEGER))]);
Decorate(PrimitiveLiteral ::= FLOAT) -> SemanticConstant
	:= (SemanticConstant[value=new Float(TokenWorth(FLOAT))]);
Decorate(PrimitiveLiteral ::= STRING) -> SemanticConstant
	:= (SemanticConstant[value=new String(TokenWorth(STRING))]);
+Decorate(PrimitiveLiteral ::= Symbol) -> SemanticConstant
+	:= Decorate(Symbol);

Decorate(TypeKeyword ::= "void") -> SemanticTypeConstant
	:= (SemanticTypeConstant[value=Void]);
Decorate(TypeKeyword ::= "bool") -> SemanticTypeConstant
	:= (SemanticTypeConstant[value=Boolean]);
+Decorate(TypeKeyword ::= "symbol") -> SemanticTypeConstant
+	:= (SemanticTypeConstant[value=Symbol]);
Decorate(TypeKeyword ::= "int") -> SemanticTypeConstant
	:= (SemanticTypeConstant[value=Integer]);
Decorate(TypeKeyword ::= "float") -> SemanticTypeConstant
	:= (SemanticTypeConstant[value=Float]);
Decorate(TypeKeyword ::= "str") -> SemanticTypeConstant
	:= (SemanticTypeConstant[value=String]);

Decorate(TypeUnit ::= PrimitiveLiteral) -> SemanticTypeConstant
	:= (SemanticTypeConstant[value=ToType(Decorate(PrimitiveLiteral).value)]);

Decorate(ExpressionUnit ::= PrimitiveLiteral) -> SemanticConstant
	:= Decorate(PrimitiveLiteral);
```

## Semantics
```diff
-SemanticConstant[value: Null | Boolean          | Number | String]
+SemanticConstant[value: Null | Boolean | Symbol | Number | String]
	::= ();
```

## Data Types
```diff
 - [Never](#never)
 - [Void](#void)
 - [Null](#null)
 - [Boolean](#boolean)
+- [Symbol](#symbol)
 - [Integer](#integer)
 - [Float](#float)
 - [String](#string)
 - [Object](#object)
 - [Unknown](#unknown)

+#### Symbol
+The **Symbol** type contains programmer-defined values whose implementations are hidden.
+They may only be referred to by name.
+The meaning of each Symbol value may be specified by the programmer.
```

## ValueOf
```diff
-Or<Null, Boolean,         Number, String> ValueOf(SemanticConstant const) :=
+Or<Null, Boolean, Symbol, Number, String> ValueOf(SemanticConstant const) :=
	1. *Return:* `const.value`.
;
```

## Build
Note: Symbols will be implemented as unsigned int16 values in the virtual machine.
```diff
+Sequence<Instruction> Build(Symbol a) :=
+	1. *Return:* ["Push `a` onto the operand stack."].
+;
```
