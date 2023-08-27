Optional entries in tuple and record types.

# Discussion

## Optional Tuple Items
A tuple type may have optional items, meaning that a tuple assigned to it might or might not have those items.
```cp
let unfixed x: [str, int, ?:bool] = ["hello", 42];
x = ["hello", 42, true];
```

Notice the new syntax in the type signature: The token `?:` indicates the item is optional. The symbol `?:` may be read aloud as “maybe”. In a tuple type, all optional items *must* come after all required items; it’s a syntax error otherwise.
```cp
type X = [str, ?:bool, int]; %> ParseError
```

## Optional Record Properties
A record type may have optional properties, meaning that a record assigned to it might or might not have those properties.
```cp
let unfixed y: [firstname: str, middlename?: str, lastname: str] = [
	firstname= "Martha",
	lastname=  "Dandridge",
];
y = [
	firstname=  "Martha",
	lastname=   "Washington",
	middlename= "Dandridge",
];
```

Notice the new syntax in the type signature: The token `?:` indicates the property is optional. The symbol `?:` may be read aloud as “maybe”. In a record type, optional properties *may* come before required properties.

## Type Compatibility
If a tuple has optional items, its count (its length) is a range of integers rather than a single integer. For example, the count of type `[str, int, ?:bool]` is `2 | 3`. When checking tuple subtypes, the minimum of the assigned (the sub) count must be at least equal to the minimum of the assignee (the super) count. E.g., a tuple of count `3 | 4 | 5` is assignble to a tuple of count `2 | 3 | 4 | 5 | 6` only because the minimum of the former (`3`) is greater than or equal to the minimum of the latter (`2`). The maximums (`5` and `6` respectively) are not considered when checking subtypes. Tuple types are checked per entry as usual, ignoring optionality.

For record types, count is computed the same way (as a range of integers, by minimum). Further, optionality is considered. Required properties are assignable to their optional counterparts, but not the other way around. E.g., `[a: str, b?: int, c: bool]` is a subtype of `[a?: str, b?: int, c: bool]`, but is *not* a subtype of `[a?: str, b: int, c: bool]`.

## VoidError
A new type of runtime error, `VoidError`, is thrown when the runtime virtual machine attempts to access a value that does not exist. This happens when an expression with no value is operated on, given as an entry to a collection (such as a tuple or record), given as a function argument, or assigned to a variable. The standard code for this error is *3100* and the standard message is *"Value is undefined."*.

## Voidish Entries
If an entry in a tuple or record is optional, its type is automatically unioned with `void` (#70). This prevents careless assignment at compile-time.
```cp
let x: [str, int, ?:bool] = ["hello", 42];
% typeof x.2 == "bool | void";

let x_2: bool = x.2; %> TypeError: `bool | void` not assignable to `bool`.
```
Updating the variable type satisfies the compiler, but could still result in a runtime VoidError.
```cp
let x_2: bool | void = x.2; % could throw a VoidError at runtime!
```
If there exists a value to which `x.2` points, then the variable assignment is successful. However, in this case, a VoidError is thrown because `x.2` doesn’t point to an actual value.

One way to work with optional properties safely and effectively is to use conditional checks before accessing them.
```cp
let x_2: bool = if x.count >= 3 then x.2 else false;
```
Since `x.count` is at least 3, the type-checker knows `x.2` must be of type `bool` and not `bool | void`.

```cp
let y: [firstname: str, middlename?: str, lastname?: str] = [
	firstname= "Martha",
	lastname=  "Dandridge",
];
let y_middlename: str = if y.count >=2 then y.middlename else ""; % TypeError: `str | void` not assinable to `str`.
```
For optional record properties, checking the count isn’t ideal, since there’s no guarantee of the order of properties. Even if the count of `y` is at least 2, the compiler can’t be certain that `y.middlename` is set, so the type of that expression is still `str | void`.

# Specification

## Lexicon
```diff
Punctuator :::=
	// annotation
+		| "?:"
;
```

## Syntax
```diff
-PropertyType
-	::= Word ":" Type;

+EntryType<Named, Optional>
+	::= <Named+>(Word . <Optional->":") <Optional+>"?:" Type;

+ItemsType ::=
+	|  EntryType<-Named><-Optional># ","?
+	| (EntryType<-Named><-Optional># ",")? EntryType<-Named><+Optional># ","?
+;

+PropertiesType
+	::= EntryType<+Named><-Optional, +Optional># ","?;

-TypeTupleLiteral  ::= "[" ","? Type#         ","? "]";
-TypeRecordLiteral ::= "[" ","? PropertyType# ","? "]";
+TypeTupleLiteral  ::= "[" ","? ItemsType          "]";
+TypeRecordLiteral ::= "[" ","? PropertiesType     "]";
```

## Semantics
```diff
+SemanticItemType[optional: Boolean]
+	::= SemanticType;

-SemanticPropertyType
+SemanticPropertyType[optional: Boolean]
	::= SemanticKey SemanticType;

SemanticTypeTuple
-	::= SemanticType*;
+	::= SemanticItemType*;
```

## Decorate
```diff
-Decorate(PropertyType ::= Word ":" Type) -> SemanticPropertyType
-	:= (SemanticPropertyType
-		Decorate(Word)
-		Decorate(Type)
-	);

+Decorate(EntryType ::= Type) -> SemanticItemType
+	:= (SemanticItemType[optional=false] Decorate(Type));
+Decorate(EntryType_Optional ::= "?:" Type) -> SemanticItemType
+	:= (SemanticItemType[optional=true] Decorate(Type));
+Decorate(EntryType_Named ::= Word ":" Type) -> SemanticPropertyType
+	:= (SemanticPropertyType[optional=false]
+		Decorate(Word)
+		Decorate(Type)
+	);
+Decorate(EntryType_Named_Optional ::= Word "?:" Type) -> SemanticPropertyType
+	:= (SemanticPropertyType[optional=true]
+		Decorate(Word)
+		Decorate(Type)
+	);

+Decorate(ItemsType ::= EntryType# ","?) -> Sequence<SemanticItemType>
+	:= ParseList(EntryType, SemanticItemType);
+Decorate(ItemsType ::= EntryType_Optional# ","?) -> Sequence<SemanticItemType>
+	:= ParseList(EntryType_Optional, SemanticItemType);
+Decorate(ItemsType ::= EntryType# "," EntryType_Optional# ","?) -> Sequence<SemanticItemType>
+	:= [
+		...ParseList(EntryType,          SemanticItemType),
+		...ParseList(EntryType_Optional, SemanticItemType),
+	];

+Decorate(PropertiesType ::= EntryType_Named# ","?) -> Sequence<SemanticPropertyType>
+	:= ParseList(EntryType_Named, SemanticPropertyType);
+Decorate(PropertiesType ::= EntryType_Named_Optional# ","?) -> Sequence<SemanticPropertyType>
+	:= ParseList(EntryType_Named_Optional, SemanticPropertyType);

-Decorate(TypeTupleLiteral ::= "[" ","? TypeTupleLiteral__0__List ","? "]") -> SemanticTypeTuple
+Decorate(TypeTupleLiteral ::= "[" ","? ItemsType                      "]") -> SemanticTypeTuple
	:= (SemanticTypeTuple
-		...Decorate(TypeTupleLiteral__0__List)
+		...Decorate(ItemsType)
	);

-	Decorate(TypeTupleLiteral__0__List ::= Type) -> Sequence<SemanticType>
-		:= [Decorate(Type)];
-	Decorate(TypeTupleLiteral__0__List ::= TypeTupleLiteral__0__List "," Type) -> Sequence<SemanticType>
-		:= [
-			...Decorate(TypeTupleLiteral__0__List),
-			Decorate(Type),
-		];

-Decorate(TypeRecordLiteral ::= "[" ","? TypeRecordLiteral__0__List ","? "]") -> SemanticTypeRecord
+Decorate(TypeRecordLiteral ::= "[" ","? PropertiesType                  "]") -> SemanticTypeRecord
	:= (SemanticTypeRecord
-		...Decorate(TypeRecordLiteral__0__List)
+		...Decorate(PropertiesType)
	);

-	Decorate(TypeRecordLiteral__0__List ::= PropertyType) -> Sequence<SemanticPropertyType>
-		:= [Decorate(PropertyType)];
-	Decorate(TypeRecordLiteral__0__List ::= TypeRecordLiteral__0__List "," PropertyType) -> Sequence<SemanticPropertyType>
-		:= [
-			...Decorate(TypeRecordLiteral__0__List),
-			Decorate(PropertyType),
-		];
```

## ParseList
`ParseList` is an attribute grammar generator. It takes the parameters `ParseNode` and `ASTNode`, and returns a generic `Decorate` attribute.
```diff
+ParseList(ParseNode, ASTNode)(ParseNode__List ::= ParseNode) -> Sequence<ASTNode>
+	:= [Decorate(ParseNode)];
+ParseList(ParseNode, ASTNode)(ParseNode__List ::= ParseNode__List ","? ParseNode) -> Sequence<ASTNode>
+	:= [
+		...Decorate(ParseNode__List),
+		Decorate(ParseNode),
+	];
```

## TypeOf
```diff
Type! TypeOfUnassessed(SemanticAccess access) :=
#	...
	5. *If* `accessor` is a SemanticIndex:
		...
		6. *If* *UnwrapAffirm:* `Subtype(base_type, Tuple)` *and* `i` is an index in `base_type`:
-			1. *Return:* the type accessed at index `i` in `base_type`.
+			1. *Let* `typedatum` be the item accessed at index `i` in `base_type`.
+			2. *Return:* *UnwrapAffirm:* `MaybeOptional(typedatum)`.
		...
	6. *If* `accessor` is a SemanticKey:
		...
		2. *If* *UnwrapAffirm:* `Subtype(base_type, Record)` *and* `id` is a key in `base_type`:
-			1. *Return:* the type accessed at key `id` in `base_type`.
+			1. *Let* `typedatum` be the item accessed at key `id` in `base_type`.
+			2. *Return:* *UnwrapAffirm:* `MaybeOptional(typedatum)`.
		...
	7. *Assert:* `accessor` is a SemanticExpression.
	...
	9. *If* *UnwrapAffirm:* `Subtype(base_type, Tuple)`:
		1. *If* `accessor_type` is a TypeConstant:
			...
			2. *If* `i` is a instance of `Integer`:
				1. *If* `i` is an index in `base_type`:
-					1. *Return:* the type accessed at index `i` in `base_type`.
+					1. *Let* `typedatum` be the item accessed at index `i` in `base_type`.
+					2. *Return:* *UnwrapAffirm:* `MaybeOptional(typedatum)`.
				...
		...
	...
;

+Type MaybeOptional([Type type, Boolean optional] typedatum) :=
+	1. *Let* `type` be `typedatum.type`.
+	2. *If*: `typedatum.optional` is `true`:
+		1. *Return:* `Union(type, Void)`.
+	3. *Return:* `type`.
+;
```

## Assess
```diff
Type Assess(SemanticTypeTuple tupletype) :=
	1. *Let* `data` be a new Sequence.
	2. *For each* `itemtype` in `tupletype`:
		1. *Assert:* `itemtype.children.count` is 1.
-		2. *Let* `type` be *UnwrapAffirm:* `Assess(itemtype.children.0)`.
-		3. Push `type` to `data`.
+		2. *Let* `typedatum` be a new Structure [
+			type:     *UnwrapAffirm:* `Assess(itemtype.children.0)`,
+			optional: `itemtype.optional`,
+		].
+		3. Push `typedatum` to `data`.
	3. *Return:* a subtype of `Tuple` whose items are `data`.
;

Type Assess(SemanticTypeRecord recordtype) :=
	1. *Let* `data` be a new Structure.
	2. *For each* `propertytype` in `recordtype`:
		1. *Assert:* `propertytype.children.count` is 2.
-		2. *Let* `type` be *UnwrapAffirm:* `Assess(propertytype.children.1)`.
-		3. Set the property `propertytype.children.0.id` on `data` to the value `type`.
+		2. *Let* `typedatum` be a new Structure [
+			type:     *UnwrapAffirm:* `Assess(propertytype.children.1)`,
+			optional: `propertytype.optional`,
+		].
+		3. Set the property `propertytype.children.0.id` on `data` to the value `typedatum`.
	3. *Return:* a subtype of `Record` whose properties are `data`.
;
```

## Subtype
```diff
Boolean Subtype(Type a, Type b) :=
#	...
	10. *If* `Equal(a, Tuple)` *and* `Equal(b, Tuple)`:
		1. *Let* `seq_a` be a Sequence whose items are exactly the items in `a`.
		2. *Let* `seq_b` be a Sequence whose items are exactly the items in `b`.
+		3. *Let* `seq_a_req` be a filtering of `seq_a` for each `ia` such that `ia.optional` is `false`.
+		4. *Let* `seq_b_req` be a filtering of `seq_b` for each `ib` such that `ib.optional` is `false`.
-		5. *If* `seq_a.count` is less than `seq_b.count`:
+		5. *If* `seq_a_req.count` is less than `seq_b_req.count`:
			1. *Return:* `false`.
		6. *For index* `i` in `seq_b`:
+			1. *If* `seq_b[i].optional` is `false`:
+				1. *Assert:* `seq_a[i]` is set *and* `seq_a[i].optional` is `false`.
-			2. *If* *UnwrapAffirm:* `Subtype(seq_a[i], seq_b[i])` is `false`:
+			2. *If* `seq_a[i]` is set *and* *UnwrapAffirm:* `Subtype(seq_a[i].type, seq_b[i].type)` is `false`:
				1. *Return:* `false`.
		7. *Return:* `true`.
	11. *If* `Equal(a, Record)` *and* `Equal(b, Record)`:
		1. *Let* `struct_a` be a Structure whose properties are exactly the properties in `a`.
		2. *Let* `struct_b` be a Structure whose properties are exactly the properties in `b`.
+		3. *Let* `struct_a_req` be a filtering of `struct_a`’s values for each `va` such that `va.optional` is `false`.
+		4. *Let* `struct_b_req` be a filtering of `struct_b`’s values for each `vb` such that `vb.optional` is `false`.
-		5. *If* `struct_a.count` is less than `struct_b.count`:
+		5. *If* `struct_a_req.count` is less than `struct_b_req.count`:
			1. *Return:* `false`.
		6. *For key* `k` in `struct_b`:
-			1. *If* `struct_a[k]` is not set:
-				1. *Return:* `false`.
+			1. *If* `struct_b[k].optional` is `false`:
+				1. *If* `struct_a[k]` is not set *or* `struct_a[k].optional` is `true`:
+					1. *Return:* `false`.
-			2. *If* *UnwrapAffirm:* `Subtype(struct_a[k], struct_b[k])` is `false`:
+			2. *If* `struct_a[k]` is set *and* *UnwrapAffirm:* `Subtype(struct_a[k].type, struct_b[k].type)` is `false`:
				1. *Return:* `false`.
		7. *Return:* `true`.
```

# Checklist
- [x] add optional item/property syntax to tuple/record types respectively
- [x] `Assess(SemanticType{Tuple,Record})` AST nodes
- [x] update `Subtype` algorithm
- [x] update `TypeOfUnassessed(SemanticAccess)`
