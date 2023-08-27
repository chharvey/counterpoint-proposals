Collection literals include list literals, record literals, and mapping literals. This issue is for syntax only, and does not provide a specification for semantic analysis (including types).

# Description

## List Literals
List literals are syntax expressions that contain a list of comma-separated expressions. Nesting is allowed.
```
[1, 2.0, "three", [4]]
```

List literals can be of type Tuple. Tuple types are written as a list of type expressions. The list above has the following type:
```
[int, float, str, [int]]
```
We can declare a variable with a tuple type and then assign it a list literal.
```
let my_tuple : [int, float, str] = [3, 4.0, "five"];
```

## Record Literals
Record literals are expressions that conatin a list of properties. **Properties** are key-value pairs. **Keys** are keywords or identifiers, and **values** are expressions. The key and value within a property are delimited by an equals sign. Properties are separated by commas. Nesting is allowed.
```
[
	first=  "who",
	second= ["what"],
	third=  [i= "don’t know"],
]
```
Keys may be reserved keywords, not just restricted to identifiers. This is because the record key will always be lexically bound to the record — it will never stand alone, so there’s no risk of syntax error.
```
[
	let=   "to initialize a variable",
	is=    "referential identity",
	int=   "the Integer type",
	false= "the negative boolean value",
]
```
Conventionally, whitespace is omitted between the key name and the equals sign delimiter `=`. This practice helps programmers differentiate between record properties and variable declarations/assignments.

Records have literal types as well. They are similar to record values, except that the colon `:` is used as the key–value delimiter, and the property values are replaced with types. Notice how the properties may be written in any order.
```
type StyleMap = [
	fontWeight: int,
	fontStyle:  "normal" | "italic" | "oblique",
	fontSize:   float,
	fontFamily: string,
];
let my_styles: StyleMap = [
	fontFamily= "sans-serif",
	fontSize=   1.25;
	fontStyle=  "oblique",
	fontWeight= 400,
];
```

## Mapping Literals
Mapping literals are expressions that conain a list of cases. **Cases** are correspondances from one or more antecedents to one consequent. **Antecedents** and **consequents** are expressions. The antecedent(s) and consequent within a case are delimited with the symbol `|->`. Antecedents within a case, and cases themselves, are separated by commas. Nesting is allowed.
```
[
	1        |-> "who",
	2, "2nd" |-> ["what"],
	1 + 2    |-> ["i" |-> ["don’t" |-> "know"]],
]
```

There is currently no syntax production for the types of mapping literals, so we must use `Object` instead.
```
let numbers: Object = [
	"one", "uno", "eine" |-> 1,
	"two", "dos", "zwei" |-> 2,
];
```

# Specification

## Lexical Grammar
```diff
Punctuator :::=
	// grouping
+		| "[" | "]"
+		| "," | "|->"
;
```

## Syntactic Grammar
```diff
+KEYWORD   ::= [./lexicon.ebnf#Keyword];
IDENTIFIER ::= [./lexicon.ebnf#Identifier];

+Word ::=
+	| KEYWORD
+	| IDENTIFIER
+;

+TypeProperty
+	::= Word ":" Type;

+TypeTupleLiteral  ::= "[" ","? Type#         ","? "]";
+TypeRecordLiteral ::= "[" ","? TypeProperty# ","? "]";

TypeUnit ::=
+	| "[" "]"
+	| TypeTupleLiteral
+	| TypeRecordLiteral
;

+Property ::= Word        "="   Expression;
+Case     ::= Expression# "|->" Expression;

+ListLiteral    ::= "[" ","? Expression# ","? "]";
+RecordLiteral  ::= "[" ","? Property#   ","? "]";
+MappingLiteral ::= "[" ","? Case#       ","? "]";

ExpressionUnit ::=
+	| "[" "]"
	| IDENTIFIER
	| PrimitiveLiteral
	| StringTemplate
+	| ListLiteral
+	| RecordLiteral
+	| MappingLiteral
	| "(" Expression ")"
;
```

## Semantic Schema
```diff
SemanticType =:=
	| SemanticTypeConstant
	| SemanticTypeAlias
+	| SemanticTypeEmptyCollection
+	| SemanticTypeList
+	| SemanticTypeRecord
	| SemanticTypeOperation
;

SemanticExpression =:=
	| SemanticConstant
	| SemanticVariable
	| SemanticTemplate
+	| SemanticEmptyCollection
+	| SemanticList
+	| SemanticRecord
+	| SemanticMapping
	| SemanticOperation
;

+SemanticKey[id: RealNumber]
+	::= ();

+SemanticTypeProperty
+	::= SemanticKey SemanticType;

+SemanticTypeEmptyCollection ::= ();
+SemanticTypeList            ::= SemanticType+;
+SemanticTypeRecord          ::= SemanticTypeProperty+;

+SemanticProperty
+	::= SemanticKey SemanticExpression;

+SemanticCase
+	::= SemanticExpression SemanticExpression+;

+SemanticEmptyCollection ::= ();
+SemanticList            ::= SemanticExpression+;
+SemanticRecord          ::= SemanticProperty+;
+SemanticMapping         ::= SemanticCase+;
```

## Decorate
```
Decorate(Word ::= IDENTIFIER) -> SemanticKey
	:= (SemanticKey[id=TokenWorth(IDENTIFIER)]);
Decorate(Word ::= KEYWORD) -> SemanticKey
	:= (SemanticKey[id=TokenWorth(KEYWORD)]);

Decorate(TypeProperty ::= Word ":" Type) -> SemanticTypeProperty
	:= (SemanticTypeProperty
		Decorate(Word)
		Decorate(Type)
	);

Decorate(TypeTupleLiteral ::= "[" ","? TypeTupleLiteral__0__List ","? "]") -> SemanticTypeList
	:= (SemanticTypeList
		...Decorate(TypeTupleLiteral__0__List)
	);

	Decorate(TypeTupleLiteral__0__List ::= Type) -> Sequence<SemanticType>
		:= [Decorate(Type)];
	Decorate(TypeTupleLiteral__0__List ::= TypeTupleLiteral__0__List "," Type) -> Sequence<SemanticType>
		:= [
			...Decorate(TypeTupleLiteral__0__List),
			Decorate(Type),
		];

Decorate(TypeRecordLiteral ::= "[" ","? TypeRecordLiteral__0__List ","? "]") -> SemanticTypeRecord
	:= (SemanticTypeRecord
		...Decorate(TypeRecordLiteral__0__List)
	);

	Decorate(TypeRecordLiteral__0__List ::= TypeProperty) -> Sequence<SemanticTypeProperty>
		:= [Decorate(TypeProperty)];
	Decorate(TypeRecordLiteral__0__List ::= TypeRecordLiteral__0__List "," TypeProperty) -> Sequence<SemanticTypeProperty>
		:= [
			...Decorate(TypeRecordLiteral__0__List),
			Decorate(TypeProperty),
		];

Decorate(TypeUnit ::= "[" "]") -> SemanticTypeEmptyCollection
	:= (SemanticTypeEmptyCollection);
Decorate(TypeUnit ::= TypeTupleLiteral) -> SemanticTypeList
	:= Decorate(TypeTupleLiteral);
Decorate(TypeUnit ::= TypeRecordLiteral) -> SemanticTypeRecord
	:= Decorate(TypeRecordLiteral);

Decorate(Property ::= Word "=" Expression) -> SemanticProperty
	:= (SemanticProperty
		Decorate(Word)
		Decorate(Expression)
	);

Decorate(Case ::= Case__0__List "|->" Expression) -> SemanticCase
	:= (SemanticCase
		...Decorate(Case__0__List)
		Decorate(Expression)
	);

	Decorate(Case__0__List ::= Expression) -> Sequence<SemanticExpression>
		:= [Decorate(Expression)];
	Decorate(Case__0__List ::= Case__0__List "," Expression) -> Sequence<SemanticExpression>
		:= [
			...Decorate(Case__0__List),
			Decorate(Expression),
		];

Decorate(ListLiteral ::= "[" ","? ListLiteral__0__List ","? "]") -> SemanticList
	:= (SemanticList
		...Decorate(ListLiteral__0__List)
	);

	Decorate(ListLiteral__0__List ::= Expression) -> Sequence<SemanticExpression>
		:= [Decorate(Expression)];
	Decorate(ListLiteral__0__List ::= ListLiteral__0__List "," Expression) -> Sequence<SemanticExpression>
		:= [
			...Decorate(ListLiteral__0__List),
			Decorate(Expression),
		];

Decorate(RecordLiteral ::= "[" ","? RecordLiteral__0__List ","? "]") -> SemanticRecord
	:= (SemanticRecord
		...Decorate(RecordLiteral__0__List)
	);

	Decorate(RecordLiteral__0__List ::= Property) -> Sequence<SemanticProperty>
		:= [Decorate(Property)];
	Decorate(RecordLiteral__0__List ::= RecordLiteral__0__List "," Property) -> Sequence<SemanticProperty>
		:= [
			...Decorate(RecordLiteral__0__List),
			Decorate(Property),
		];

Decorate(MappingLiteral ::= "[" ","? MappingLiteral__0__List ","? "]") -> SemanticMapping
	:= (SemanticMapping
		...Decorate(MappingLiteral__0__List)
	);

	Decorate(MappingLiteral__0__List ::= Case) -> Sequence<SemanticCase>
		:= [Decorate(Case)];
	Decorate(MappingLiteral__0__List ::= MappingLiteral__0__List "," Case) -> Sequence<SemanticCase>
		:= [
			...Decorate(MappingLiteral__0__List),
			Decorate(Case),
		];

Decorate(ExpressionUnit ::= "[" "]") -> SemanticEmptyCollection
	:= (SemanticEmptyCollection);
Decorate(ExpressionUnit ::= ListLiteral) -> SemanticList
	:= Decorate(ListLiteral);
Decorate(ExpressionUnit ::= RecordLiteral) -> SemanticRecord
	:= Decorate(RecordLiteral);
Decorate(ExpressionUnit ::= MappingLiteral) -> SemanticMapping
	:= Decorate(MappingLiteral);
```
