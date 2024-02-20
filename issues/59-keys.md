Keys are global primitive values without intrinsic semantics.

# Discussion
Keys are values whose implementations are hidden.
```cp
.WATER; % a key
.FIRE;  % another key
```
Key names are programmer-defined identifiers, and always follow a dot. They’re conventionally written in MACRO_CASE, but sometimes also in snake_case.

The values of Keys are the only of their kind, and their implementations are unexposed. Keys can be thought of as integers or strings, but can’t be operated on as such and do not take up as much space. Keys are also not assignable to the `int` or `str` types. Like all values, keys can be assigned to types `Object` and `unknown`.
```cp
let greet: Object = .hello_world;
```

Key names may be any keyword or identifier, starting with a letter or underscore, and subsequently containing letters, underscores, and/or digits. Key names may also be Unicode names, but this is not recommended as it hinders readability. As with Unicode identifiers, there are no escapes.
```cp
let greet: unknown = .'¡héllö wòrld!';

.'\u{24}3.99' != .'$3.99';
```

Other than `Object` and `unknown`, Keys are assignable to their unit types. (Unit types are types containing only one value — also referred to as “constant types”.)
```cp
let earth: .EARTH = .EARTH;
```

Key unit types can be unioned to form an enumeration of values.
```cp
type Element = .WATER | .EARTH | .FIRE | .AIR;
let var el: Element = .FIRE;
set el = .AIR;
set el = .AETHER; %> TypeError
```
All Key unit types are disjoint, so their intersection is always `never`.
```cp
type Impossible = .WATER & .EARTH; % type `never`
```

Counterpoint has a designated `key` type, which is conceptually the infinite union of all Key values. Any Key value is assignable to the `key` type, and *only* Key values are assignable to this type.
```cp
let var el: key = .FIRE;
set el = .AIR;
set el = .AETHER;
set el = 42;         %> TypeError
set el = "a string"; %> TypeError
```

Key values are always truthy and always nonempty.
```cp
!(.WATER) == false; % `!.WATER` is a parse error
?(.FIRE)  == false; % `?.FIRE`  is a parse error

.EARTH || .AIR == .EARTH;
.EARTH && .AIR == .AIR;
```
(Note: The Keys above must be wrapped in parentheses, because the tokens `!.` and `?.` are accessor operators. Or we could use whitespace to separate the operator from the Key.)
```cp
! .WATER == false; % also acceptable, but maybe less readable
? .FIRE  == false; % also acceptable, but maybe less readable
```

Keys are equal if and only if their names are exactly the same. Keys are identical if and only if they are equal.
```cp
.LIGHT   !=  .DARK;
.NEUTRAL === .NEUTRAL;
.NEUTRAL !=  .neutral;
```

As primitive values, Key values are value objects rather than reference objects.

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
+		| "key"
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

+Key
+	::= "." Word;

PrimitiveLiteral ::=
	| "null"
	| "false"
	| "true"
	| INTEGER
	| FLOAT
	| STRING
+	| Key
;

TypeKeyword ::=
	| "void"
	| "bool"
+	| "key"
	| "int"
	| "float"
	| "str"
;
```

## Decorate
```diff
+Decorate(Key ::= "." Word) -> SemanticConstant
+	:= (SemanticConstant[value=new Key(Decorate(Word).id)]);

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
+Decorate(PrimitiveLiteral ::= Key) -> SemanticConstant
+	:= Decorate(Key);

Decorate(TypeKeyword ::= "void") -> SemanticTypeConstant
	:= (SemanticTypeConstant[value=Void]);
Decorate(TypeKeyword ::= "bool") -> SemanticTypeConstant
	:= (SemanticTypeConstant[value=Boolean]);
+Decorate(TypeKeyword ::= "key") -> SemanticTypeConstant
+	:= (SemanticTypeConstant[value=Key]);
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
-SemanticConstant[value: Null | Boolean |       Number | String]
+SemanticConstant[value: Null | Boolean | Key | Number | String]
	::= ();
```

## Data Types
```diff
 - [Never](#never)
 - [Void](#void)
 - [Null](#null)
 - [Boolean](#boolean)
+- [Key](#key)
 - [Integer](#integer)
 - [Float](#float)
 - [String](#string)
 - [Object](#object)
 - [Unknown](#unknown)

+#### Key
+The **Key** type contains programmer-defined values whose implementations are hidden.
+They may only be referred to by name.
+The meaning of each Key value may be specified by the programmer.
```

## ValueOf
```diff
-Or<Null, Boolean,      Number, String> ValueOf(SemanticConstant const) :=
+Or<Null, Boolean, Key, Number, String> ValueOf(SemanticConstant const) :=
	1. *Return:* `const.value`.
;
```

## Build
Note: Keys will be implemented as int32 values in the virtual machine.
```diff
+Sequence<Instruction> Build(Key a) :=
+	1. *Return:* ["Push `a` onto the operand stack."].
+;
```
