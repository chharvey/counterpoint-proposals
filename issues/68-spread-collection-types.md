Spread syntax for tuple types and record types.

# Discussion
Spread for tuple types and record types is parallel to spread for tuples and records (#64). Tuple types may be spread into other tuple types, and record types may be spread into other record types. Spreading a collection type is shorthand for injecting each of that type’s members.
```cp
type CDE = [str, str, str];
type ABCDEFG = [str, str, #CDE, str, str];
let abcdefg: ABCDEFG = ["a", "b", "c", "d", "e", "f", "g"];
```
```cp
type Font = [
	family: str,
	weight: "bold" | "normal",
	style:  "italic" | "normal",
];
type BetterFont = [
	weight: int,
	##Font,
	style:  "italic" | "oblique" | "normal",
	color:  [float, float, float, float],
]; %% [
	family: str,
	weight: "bold" | "normal",
	style: "italic" | "oblique" | "normal",
	color: [float, float, float, float],
] %%
```

Note that spreading record types is not the same as intersecting them. The former overrides properties in source order, whereas the latter takes the union of the properties but intersects the property types.
```cp
type FontIntersection = [
	family: str,
	weight: "bold" | "normal",
	style:  "italic" | "normal",
] & [
	weight: int,
	style:  "italic" | "oblique" | "normal",
	color:  [float, float, float, float],
]; %% [
	family: str,
	weight: never,
	style:  "italic" | "normal",
	color:  [float, float, float, float],
] %%
```

# Specification

## Syntax
```diff
-TypeTupleLiteral  ::= "[" ","?              Type #         ","? "]";
-TypeRecordLiteral ::= "[" ","?              PropertyType # ","? "]";
+TypeTupleLiteral  ::= "[" ","? ("#"  Type | Type)#         ","? "]";
+TypeRecordLiteral ::= "[" ","? ("##" Type | PropertyType)# ","? "]";
```

## Semantics
```diff
SemanticSpread[arity: 1 | 2 | 3]  ::= Expression;
+SemanticSpreadType[arity: 1 | 2] ::= Type;

-SemanticTypeList   ::= SemanticType+;
-SemanticTypeRecord ::= SemanticPropertyType+;
+SemanticTypeList   ::= (SemanticType         | SemanticSpreadType[arity: 1])+;
+SemanticTypeRecord ::= (SemanticPropertyType | SemanticSpreadType[arity: 2])+;
```

## Decorate
```diff
Decorate(TypeTupleLiteral ::= "[" ","? TypeTupleLiteral__0__List ","? "]") -> SemanticTypeList
	:= (SemanticTypeList
		...Decorate(TypeTupleLiteral__0__List)
	);

+	Decorate(TypeTupleLiteral__0__List ::= "#" Type) -> Sequence<SemanticSpreadType>
+		:= [(SemanticSpreadType[arity=1] Decorate(Type))];
	Decorate(TypeTupleLiteral__0__List ::= Type) -> Sequence<SemanticType>
		:= [Decorate(Type)];
+	Decorate(TypeTupleLiteral__0__List ::= TypeTupleLiteral__0__List "," "#" Type) -> Sequence<SemanticType | SemanticSpreadType>
+		:= [
+			...Decorate(TypeTupleLiteral__0__List),
+			(SemanticSpreadType[arity=1] Decorate(Type)),
+		];
-	Decorate(TypeTupleLiteral__0__List ::= TypeTupleLiteral__0__List "," Type) -> Sequence<SemanticType>
+	Decorate(TypeTupleLiteral__0__List ::= TypeTupleLiteral__0__List "," Type) -> Sequence<SemanticType | SemanticSpreadType>
		:= [
			...Decorate(TypeTupleLiteral__0__List),
			Decorate(Type),
		];

Decorate(TypeRecordLiteral ::= "[" ","? TypeRecordLiteral__0__List ","? "]") -> SemanticTypeRecord
	:= (SemanticTypeRecord
		...Decorate(TypeRecordLiteral__0__List)
	);

+	Decorate(TypeRecordLiteral__0__List ::= "##" Type) -> Sequence<SemanticSpreadType>
+		:= [(SemanticSpreadType[arity=2] Decorate(Type))];
	Decorate(TypeRecordLiteral__0__List ::= TypeProperty) -> Sequence<SemanticTypeProperty>
		:= [Decorate(TypeProperty)];
+	Decorate(TypeRecordLiteral__0__List ::= TypeRecordLiteral__0__List "," "##" Type) -> Sequence<SemanticTypeProperty | SemanticSpreadType>
+		:= [
+			...Decorate(TypeRecordLiteral__0__List),
+			(SemanticSpreadType[arity=2] Decorate(Type)),
+		];
-	Decorate(TypeRecordLiteral__0__List ::= TypeRecordLiteral__0__List "," TypeProperty) -> Sequence<SemanticTypeProperty>
+	Decorate(TypeRecordLiteral__0__List ::= TypeRecordLiteral__0__List "," TypeProperty) -> Sequence<SemanticTypeProperty | SemanticSpreadType>
		:= [
			...Decorate(TypeRecordLiteral__0__List),
			Decorate(TypeProperty),
		];
```
