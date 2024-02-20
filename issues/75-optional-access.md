The optional property access operator `?.` does not throw an error if the property does not exist. Follows from #71.

# Discussion

## Optional Access
The **optional access** operator `?.` acts like index/property access, except that it doesn’t throw a VoidError if the binding object doesn’t have the accessed entry; instead it produces `null`. This operator works really nicely with optional entries and the nullish type operator (#70). Thus instead of using conditional checks to test whether a collection has an entry, we can simply use optional access as a fallback. The expression `a?.b` is generally equivalent to (pseudocode) `try { produce a.b; } catch { produce null; };`.
```cp
let x: [str, int, ?:bool] = ["hello", 42];
let x_2: bool? = x?.2;                     % Produce `x.2` if it exists, else produce `null`.

let y: [firstname: str, middlename?: str, lastname?: str] = [
	firstname= "Martha",
	lastname=  "Dandridge",
];
let y_middlename: str? = y?.middlename; % Produce `y.middlename` if it exists, else produce `null`.
```
Note that if `a?.b` produces `null`, it either means that `a.b` does exist and is actually `null`, or that there is no value for the `b` optional property bound to `a`.

At *compile-time*, the type of an expression `a?.b` varies depending on the type `A` of `a`:
- If type `A` has a bound index/property `b` with type `B`, then the type of `a?.b` is `B`, which is the same as the type of `a.b`.
- If type `A` has an *optional* bound property `b` with type `B`, then the type of `a?.b` is `B?` (i.e., `B | null`).
- If type `A` is a subtype of `null`, then the type of `a?.b` is also `null`. (This allows chained optional access, noted below.)
- If type `A` is not a subtype of `null` and it has *no* bound property `b`, then attempting `a?.b` will result in a TypeError as ususal.
```cp
let a1: [str, int,    bool] = ["hello", 42, true];
let a2: [str, int, ?: bool] = ["hello", 42, true];
let a3: [str, int, ?: bool] = ["hello", 42];
let a4: [str, int]          = ["hello", 42, 4.2];
let a5: [str, int]          = ["hello", 42];
a1?.2; % type `bool`,  produces `true`
a2?.2; % type `bool?`, produces `true`
a3?.2; % type `bool?`, produces `null`
a4?.2; %> TypeError
a5?.2; %> TypeError

let n: null = null;
let o: Object = null;
n?.2; % type `null`, produces `null`
o?.2; %> TypeError
```
For computed expression access on tuples, the same rules apply. For mappings, (let the type of `a` be `Mapping.<K, V>` and the type of `b` be a subtype of `K`), then, the type of `a.[b]` is `V` and the type of `a?.[b]` is always `V?`.

The optional access does not short-circuit. That is, if `a?.b` produces null, then `a?.b.c.d` is equivalent to `((a?.b).c).d`, that is, `(null.c).d`, and will result in a TypeError. In order to chain optional access safely, we need to use it down the line, i.e `a?.b?.c?.d`.

## Claim Access
The **claim access** operator `!.` is a compile-time operator that *claims* (makes a type-assertion) that the type of the accessed optional property is not `void`. Rather than producing `null` at runtime, like optional access does, this operator simply accesses the property as normal. However, at compile-time, it tells the type-checker, “no, we’re absolutely sure this property exists and it’s not an Exception.” (Exceptions are not covered in this version).
```cp
let x: [str, int, ?:bool] = ["hello", 42, false];
let x_2: bool = x!.2;                             % Claim `x.2` definitely exists and is not `void`

let y: [firstname: str, middlename?: str, lastname?: str] = [
	firstname=  "Martha",
	middlename= "Dandridge",
	lastname=   "Washington",
];
let y_middlename: str = y!.middlename; % Claim `y.middlename` definitely exists and is not `void`.
```
However, we still need to be careful here, because this operator only bypasses the compiler, and does not affect runtime operation. If for example `x.2` was not set, then that would result in a runtime VoidError. The claim access operator `!.` should only be used if you know what you’re doing.

The rules for compile-time type-checking are similar:
- If type `A` has a bound index/property `b` with type `B`, *optional or not*, then the type of `a!.b` is *\`B\` - \`Void\`*. (`B - void` is not yet a valid type syntax.)
- If type `A` (even a subtype of `null`) has *no* bound property `b`, then attempting `a!.b` will result in a TypeError.

# Specification

## Lexicon
```diff
Punctuator :::=
	// access
+		| "?."
+		| "!."
;
```

## Syntax
```diff
PropertyAccess
-	::=  "."                (INTEGER | Word | "[" Expression "]");
+	::= ("." | "?." | "!.") (INTEGER | Word | "[" Expression "]");
```

## Semantics
```diff
-SemanticAccess
+SemanticAccess[kind: NORMAL | OPTIONAL | CLAIM]
	::= SemanticExpression (SemanticIndex | SemanticKey | SemanticExpression);
```

## Decorate
```diff
-Decorate(PropertyAccess ::=  "."                INTEGER) -> SemanticIndex
+Decorate(PropertyAccess ::= ("." | "?." | "!.") INTEGER) -> SemanticIndex
	:= (SemanticIndex
		(SemanticConstant[value=Integer(TokenWorth(INTEGER))])
	);
-Decorate(PropertyAccess ::=  "."                Word) -> SemanticKey
+Decorate(PropertyAccess ::= ("." | "?." | "!.") Word) -> SemanticKey
	:= Decorate(Word);
-Decorate(PropertyAccess ::=  "."                "[" Expression "]") -> SemanticExpression
+Decorate(PropertyAccess ::= ("." | "?." | "!.") "[" Expression "]") -> SemanticExpression
	:= Decorate(Expression);

Decorate(ExpressionCompound ::= ExpressionUnit) -> SemanticExpression
	:= Decorate(ExpressionUnit);
Decorate(ExpressionCompound ::= ExpressionCompound PropertyAccess) -> SemanticAccess
-	:= (SemanticAccess
+	:= (SemanticAccess[kind=AccessKind(PropertyAccess)]
		Decorate(ExpressionCompound)
		Decorate(PropertyAccess)
	);
```

## AccessKind
```diff
+AccessKind(PropertyAccess ::= "." (INTEGER | Word | "[" Expression "]")) -> NORMAL
+	::= NORMAL;
+AccessKind(PropertyAccess ::= "?." (INTEGER | Word | "[" Expression "]")) -> OPTIONAL
+	::= OPTIONAL;
+AccessKind(PropertyAccess ::= "!." (INTEGER | Word | "[" Expression "]")) -> CLAIM
+	::= CLAIM;
```

## TypeOf
```diff
Type! TypeOfUnassessed(SemanticAccess access) :=
#	...
	4. *Let* `base_type` be *Unwrap:* `CombineTuplesOrRecords(TypeOf(base))`.
+	5. *If* `access.kind` is `OPTIONAL`:
+		1. *If* *UnwrapAffirm:* `Subtype(base_type, Null)`:
+			1. *Return:* `base_type`.
+		2. *If* *UnwrapAffirm:* `Subtype(Null, base_type)`:
+			1. *Set* `is_nullish` to `true`.
+			2. *Set* `base_type` to `Difference(base_type, Null)`.
	6. *If* `accessor` is a SemanticIndex:
#		...
		6. *If* *UnwrapAffirm:* `Subtype(base_type, Tuple)` *and* `i` is an index in `base_type`:
			1. *Let* `entry` be the item accessed at index `i` in `base_type`.
-			2. *Return:* *UnwrapAffirm:* `MaybeOptional(entry)`.
+			2. *Set* `returned` to *UnwrapAffirm:* `MaybeOptional(access, entry)`.
		7. *Throw:* a new TypeError04 "Index {{ i }} does not exist on type {{ base_type }}.".
	7. *If* `accessor` is a SemanticKey:
		1. *Let* `id` be `accessor.id`.
		2. *If* *UnwrapAffirm:* `Subtype(base_type, Record)` *and* `id` is a key in `base_type`:
			1. *Let* `entry` be the item accessed at key `id` in `base_type`.
-			2. *Return:* *UnwrapAffirm:* `MaybeOptional(entry)`.
+			2. *Set* `returned` to *UnwrapAffirm:* `MaybeOptional(access, entry)`.
		3. *Throw:* a new TypeError04 "Property {{ id }} does not exist on type {{ base_type }}.".
#	...
	10. *If* *UnwrapAffirm:* `Subtype(base_type, Tuple)`:
		1. *If* `accessor_type` is a TypeConstant:
			1. *Let* `i` be `accessor_type.value`.
			2. *If* `i` is a instance of `Integer`:
				1. *If* `i` is an index in `base_type`:
					1. *Let* `entry` be the item accessed at index `i` in `base_type`.
-					2. *Return:* *UnwrapAffirm:* `MaybeOptional(entry)`.
+					2. *Set* `returned` to *UnwrapAffirm:* `MaybeOptional(access, entry)`.
				2. *Throw:* a new TypeError04 "Index {{ i }} does not exist on type {{ base_type }}.".
		2. *Else If* *UnwrapAffirm:* `Subtype(accessor_type, Integer)`:
-			1. *Return:* the union of all item types in `base_type`.
+			1. *Let* `type` be the union of all item types in `base_type`.
+			2. *If*: `access.kind` is `OPTIONAL`:
+				1. *Set* `returned` to `Union(type, Null)`.
+			3. *If*: `access.kind` is `CLAIM`:
+				1. *Set* `returned` to `Difference(type, Void)`.
+			4. *Set* `returned` to `type`.
		3. *Throw:* a new TypeError02 "Type {{ accessor_type }} is not a subtype of type {{ Integer }}.".
	11. *If* *UnwrapAffirm:* `Subtype(base_type, Mapping)`:
#		...
		3. *If* *UnwrapAffirm:* `Subtype(accessor_type, k)`:
-			1. *Return:* `v`.
+			1. *If*: `access.kind` is `OPTIONAL`:
+				1. *Set* `returned` to `Union(v, Null)`.
+			2. *If*: `access.kind` is `CLAIM`:
+				1. *Set* `returned` to `Difference(v, Void)`.
+			3. *Set* `returned` to `v`.
		4. *Throw:* a new TypeError02 "Type {{ accessor_type }} is not a subtype of type {{ k }}.".
+	12. *If* `returned` is set:
+		1. *If* `is_nullish` is `true`:
+			1. *Return:* `Union(returned, Null)`.
+		2. *Return*: `returned`.
	13. *Throw:* a new TypeError01.
;
```

## MaybeOptional
```diff
Type MaybeOptional(SemanticAccess access, EntryTypeStructure entry) :=
	1. *Let* `type` be `entry.type`.
+	2. *If*: `access.kind` is `CLAIM`:
+		1. *Return:* `Difference(type, Void)`.
	3. *If*: `entry.optional` is `true`:
+		1. *If*: `access.kind` is `OPTIONAL`:
+			1. *Return:* `Union(type, Null)`.
		2. *Return:* `Union(type, Void)`.
	4. *Return:* `type`.
;
```

## Assess
```diff
Object! Assess(SemanticAccess access) :=
	...
	4. *Let* `base_value` be *Unwrap:* `Assess(base)`.
+	5. *If* `access.kind` is `OPTIONAL` *and* *UnwrapAffirm:* `Equal(base_value, null)`:
+		1. *Return:* `base_value`.
	...
	6. *If* `accessor` is a SemanticIndex:
		...
	12. *If* `accessor_value` is an antecedent in `base_value`:
		...
+		3. *If*: `access.kind` is `OPTIONAL`:
+			1. *Return:* `null`.
	13. *Throw:* a new VoidError.
;
```

# Checklist
- [x] update reference docs (optional access)
- [x] add optional access expression syntax
- [x] type of optional access expressions
- [x] assess optional access expressions
- [x] update reference docs (claim access)
- [x] add claim access expression syntax
- [x] type of claim access expressions
- [x] assess claim access expressions
