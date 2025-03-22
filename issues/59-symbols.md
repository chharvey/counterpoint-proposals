Symbols are global primitive values with opaque representations and without intrinsic semantics.

# Discussion
Symbols are values whose implementations are hidden. They are only referenced by their name.
```cp
@WATER; % a symbol
@FIRE;  % another symbol
```
Symbol names are programmer-defined identifiers, and always follow an “at” sign. They’re conventionally written in MACRO_CASE, but sometimes also in snake_case.

The values of symbols are the only of their kind, and their implementations are unexposed. Symbols can be thought of as integers or strings, but can’t be operated on as such and do not take up as much space. For example, one cannot add symbols and cannot convert them to uppercase. Symbols are not assignable to the `int` or `str` types, and vice versa. Symbols are “value types”, that is, compared and copied by value rather than by reference. Like all values, symbols are assignable to type `unknown`.
```cp
let greet: unknown = @hello_world;

@hello + @world;             %> TypeError
@hello_world.toUppercase.(); %> TypeError
```

Symbol names may be any keyword or identifier, starting with a letter or underscore, and subsequently containing letters, underscores, and/or digits. Even the single underscore `_` is a valid sybmol name. Symbol names may also be Unicode names (in single quotes), but this is not recommended as it hinders readability. As with Unicode identifiers, there are no escapes. When stringified, the the resulting string is the symbol’s name, unescaped.
```cp
let greet: unknown = @'¡héllö wòrld!';

@'\u{24}3.99' != @'$3.99';

let greeting: str = """{{ greet }}""";
greeting; %== "¡héllö wòrld!"

let price: str = """{{ @'\u{24}3.99' }}""";
price; %== """'\u{24}3.99'"""
% notice the string include the symbol’s single-quotes and backslash character
```

Other than `unknown`, symbols are assignable to their unit types. (Unit types are types containing only one value — also referred to as “constant types”.)
```cp
let earth: @EARTH = @EARTH;
```

Symbol unit types can be unioned to form an enumeration of values.
```cp
type Element = @WATER | @EARTH | @FIRE | @AIR;
let var el: Element = @FIRE;
set el = @AIR;
set el = @AETHER; %> TypeError
```
All symbol unit types are disjoint, so their intersection is always `never`.
```cp
type Impossible = @WATER & @EARTH; % type `never`
```

Counterpoint has a designated `symbol` type, which is conceptually the infinite union of all potential symbol values. Any symbol value is assignable to the `symbol` type, and *only* symbol values are assignable to this type.
```cp
let var el: symbol = @FIRE;
set el = @AIR;
set el = @AETHER;
set el = 42;         %> TypeError
set el = "a string"; %> TypeError
```

Symbol values are always truthy and always nonempty.
```cp
!@WATER == false;
?@FIRE  == false;

@EARTH || @AIR == @EARTH;
@EARTH && @AIR == @AIR;
```

Symbols are equal if and only if their names are exactly the same. As “value types”, symbols are identical if and only if they are equal.
```cp
@LIGHT   !=  @DARK;
@NEUTRAL === @NEUTRAL;
@NEUTRAL !=  @neutral;
```

# Specification

## Lexicon
```diff
KeywordType :::=
	| "never"
	| "void"
	| "bool"
+	| "symbol"
	| "int"
	| "float"
	| "str"
	| "unknown"
;
```

## Syntax
```diff
Word ::=
	// operator
		| "mut"
		| "is"
		| "isnt"
		| "if"
		| "then"
		| "else"
	// storage
		| "type"
		| "let"
		| "_"
	// modifier
		| "var"
	| KEYWORD_TYPE
	| KEYWORD_VALUE
	| IDENTIFIER
;

PrimitiveLiteral ::=
	| KEYWORD_VALUE
	| INTEGER
	| FLOAT
	| STRING
+	| "@" Word
;
```

## TokenWorth
```diff
 TokenWorth(KeywordType  :::= "never")   -> RealNumber := \x80;
 TokenWorth(KeywordType  :::= "void")    -> RealNumber := \x81;
 TokenWorth(KeywordType  :::= "bool")    -> RealNumber := \x82;
+TokenWorth(KeywordType  :::= "symbol")  -> RealNumber := \x83;
-TokenWorth(KeywordType  :::= "int")     -> RealNumber := \x83;
-TokenWorth(KeywordType  :::= "float")   -> RealNumber := \x84;
-TokenWorth(KeywordType  :::= "str")     -> RealNumber := \x85;
-TokenWorth(KeywordType  :::= "unknown") -> RealNumber := \x86;
-TokenWorth(KeywordValue :::= "null")    -> RealNumber := \x87;
-TokenWorth(KeywordValue :::= "false")   -> RealNumber := \x88;
-TokenWorth(KeywordValue :::= "true")    -> RealNumber := \x89;
-TokenWorth(KeywordOther :::= "mut")     -> RealNumber := \x8a;
-TokenWorth(KeywordOther :::= "is")      -> RealNumber := \x8b;
-TokenWorth(KeywordOther :::= "isnt")    -> RealNumber := \x8c;
-TokenWorth(KeywordOther :::= "if")      -> RealNumber := \x8d;
-TokenWorth(KeywordOther :::= "then")    -> RealNumber := \x8e;
-TokenWorth(KeywordOther :::= "else")    -> RealNumber := \x8f;
-TokenWorth(KeywordOther :::= "type")    -> RealNumber := \x90;
-TokenWorth(KeywordOther :::= "let")     -> RealNumber := \x91;
-TokenWorth(KeywordOther :::= "_")       -> RealNumber := \x92;
-TokenWorth(KeywordOther :::= "var")     -> RealNumber := \x93;
+TokenWorth(KeywordType  :::= "int")     -> RealNumber := \x84;
+TokenWorth(KeywordType  :::= "float")   -> RealNumber := \x85;
+TokenWorth(KeywordType  :::= "str")     -> RealNumber := \x86;
+TokenWorth(KeywordType  :::= "unknown") -> RealNumber := \x87;
+TokenWorth(KeywordValue :::= "null")    -> RealNumber := \x88;
+TokenWorth(KeywordValue :::= "false")   -> RealNumber := \x89;
+TokenWorth(KeywordValue :::= "true")    -> RealNumber := \x8a;
+TokenWorth(KeywordOther :::= "mut")     -> RealNumber := \x8b;
+TokenWorth(KeywordOther :::= "is")      -> RealNumber := \x8c;
+TokenWorth(KeywordOther :::= "isnt")    -> RealNumber := \x8d;
+TokenWorth(KeywordOther :::= "if")      -> RealNumber := \x8e;
+TokenWorth(KeywordOther :::= "then")    -> RealNumber := \x8f;
+TokenWorth(KeywordOther :::= "else")    -> RealNumber := \x90;
+TokenWorth(KeywordOther :::= "type")    -> RealNumber := \x91;
+TokenWorth(KeywordOther :::= "let")     -> RealNumber := \x92;
+TokenWorth(KeywordOther :::= "_")       -> RealNumber := \x93;
+TokenWorth(KeywordOther :::= "var")     -> RealNumber := \x94;

 KeywordType(KEYWORD_TYPE :::= "never")   -> Type := Never;
 KeywordType(KEYWORD_TYPE :::= "void")    -> Type := Void;
 KeywordType(KEYWORD_TYPE :::= "bool")    -> Type := Boolean;
+KeywordType(KEYWORD_TYPE :::= "symbol")  -> Type := Symbol;
 KeywordType(KEYWORD_TYPE :::= "int")     -> Type := Integer;
 KeywordType(KEYWORD_TYPE :::= "float")   -> Type := Float;
 KeywordType(KEYWORD_TYPE :::= "str")     -> Type := String;
 KeywordType(KEYWORD_TYPE :::= "unknown") -> Type := Unknown;
```

## Decorate
```diff
Decorate(PrimitiveLiteral ::= KEYWORD_VALUE) -> SemanticConstant
	:= (SemanticConstant[value=KeywordValue(KEYWORD_VALUE)]);
Decorate(PrimitiveLiteral ::= INTEGER) -> SemanticConstant
	:= (SemanticConstant[value=Integer(TokenWorth(INTEGER))]);
Decorate(PrimitiveLiteral ::= FLOAT) -> SemanticConstant
	:= (SemanticConstant[value=Float(TokenWorth(FLOAT))]);
Decorate(PrimitiveLiteral ::= STRING) -> SemanticConstant
	:= (SemanticConstant[value=String(TokenWorth(STRING))]);
+Decorate(PrimitiveLiteral ::= "@" Word) -> SemanticConstant
+	:= (SemanticConstant[value=Symbol(Decorate(Word).id)]);

Decorate(TypeUnit ::= KEYWORD_TYPE) -> SemanticTypeConstant
	:= (SemanticTypeConstant[value=KeywordType(KEYWORD_TYPE)]);

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
