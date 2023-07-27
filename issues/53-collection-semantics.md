Follows from #21.

# Discussion

## New Solid Language Types

### Tuples
Tuples are fixed-size ordered lists of indexed values, with indices starting at `0`. The values in a tuple are called **items** (the actual values) or **entries** (the slots the values are stored in). The number of entries in a tuple is called its **count**. The count of a tuple is fixed and known at compile-time, as is the type of each entry in the tuple. If the tuple is mutable, the entries of the tuple may be reassigned, but only to values of the correct type.

For example, the tuple `[3, 4.0, 'five']` has an integer in the first position at index `0`, followed by a float at index `1`, followed by a string at index `2`. Entries cannot be added or removed — the count of the tuple cannot change — but entries can be reassigned: We could set the last entry to the string `'twelve'`. (Tuple mutation is not covered in this issue.)

There are no such things as “sparse tuples”: Tuples cannot have “empty slots” where items should be.

### Records
Records are fixed-size unordered lists of keyed values. Key–value pairs are called **properties**. The number of properties in a record is called its **count**. The count and types of record **entries** (the “slots” where values are stored) are also fixed and known at compile-time. Record entries cannot be added or deleted, but if the record is mutable, they can be reassigned.

For example, given the record
```cp
[
	fontFamily= 'sans-serif',
	fontSize=   1.25,
	fontStyle=  'oblique',
	fontWeight= 400,
]
```
we could reassign the `fontWeight` property a value of `700`.

### Mappings
Mappings form associations (**cases**) of values (**antecedents**) to other values (**consequents**). The antecedents are unique in that each antecedent can be associated with only one consequent. The number of cases in a mapping is called its **count**.

In this current version there is no API for manipulating mappings; they can only be set to and referenced by variables. For now, their type should be declared as `obj`, as there is not yet a more specific type available.

## Type Compatibility

### Covariance
For this issue, all tuple and record types are read-only types. Therefore, collection types are **covariant**: that is, for any types ‹S› and ‹T›, if ‹S› is a subtype of ‹T›, then ‹G›<‹S›> is a subtype of ‹G›<‹T›>, where ‹G› represents the abstract idea of a generic type or type function, taking one type parameter. (Generics / type functions are not part of this version.) Tuple and record types are covariant by entry.

For example, type `[int, bool, str]` is a subtype of type `[int | float, bool!, obj]`, since, entry-wise:
- `int` is a subtype of `int | float`
- `bool` is a subtype of `bool!`
- `str` is a subtype of `obj`

Type `[a: int, b: bool, c: str]` is a subtype of type `[b: bool!, c: obj, a: int | float]` in the same manner.

### Count
Tuples of higher count are always assignable to tuples of lower count, but never vice versa. Records having more properties are always assignable to reords having fewer, but never vice versa.
```cp
let tuple1: [int, bool] = [42, false, 'hello world'];
let record1: [a: obj, b: float] = [a= null, b= 4.2, c= true];

let tuple2: [int, bool, str] = [42, false];                   %> TypeError 1
let record2: [a: obj, b: float, c: true] = [a= null, b= 4.2]; %> TypeError 2
```
> 1. TypeError: Expression of type `[42, false]` is not assignable to type `[int, bool str]`.
> 		- TypeError: Index `2` does not exist on type `[42, false]`.
> 1. TypeError: Expression of type `[a: null, b: 4.2]` is not assignable to type `[a: obj, b: float, c: true]`.
> 		- TypeError: Property `c` does not exist on type `[a: null, b: 4.2]`.

# Specification

## TypeOf
```
Type! TypeOf(SemanticTuple tuple) :=
	1. *Let* `types` be a mapping of `tuple` for each `it` to *Unwrap:* `TypeOf(it)`.
	2. *Return:* a new Tuple type containing the items in the sequence `types`.
;

Type! TypeOf(SemanticRecord record) :=
	1. *Let* `types` be a new Structure.
	2. *For each* `property` in `record`:
		1. *Set* `types` to a new Structure [
			...types,
			`property.children.0` = *Unwrap:* `TypeOf(property.children.1)`,
		].
	3. *Return:* a new Record type containing the properties in the structure `types`.
;

Type! TypeOf(SemanticMapping mapping) :=
	1. *Let* `ant_types` be a mapping of `mapping` for each `case` to *Unwrap:* `TypeOf(case.children.0)`.
	2. *Let* `con_types` be a mapping of `mapping` for each `case` to *Unwrap:* `TypeOf(case.children.1)`.
	3. *Let* `k` be `Union(...ant_types)`.
	4. *Let* `v` be `Union(...con_types)`.
	5. *Return:* a new Mapping type whose antecedent types are `k` and whose consequent types are `v`.
;
```

## Assess
```
Tuple! Assess(SemanticTuple tuple) :=
	1. *Let* `data` be a mapping of `tuple` for each `it` to *Unwrap:* `Assess(it)`.
	2. *Return:* a new Tuple containing the items in the sequence `data`.
;

Record! Assess(SemanticRecord record) :=
	1. *Let* `data` be a new Structure.
	2. *For each* `property` in `record`:
		1. Set `data` to a new Structure [
			...data,
			`property.children.0.id`= *Unwrap:* `Assess(property.children.1)`,
		].
	3. *Return:* a new Record containing the properties in the structure `data`.
;

Mapping! Assess(SemanticMapping mapping) :=
	1. *Let* `data` be a mapping of `mapping` for each `case` to a new Sequence [
		*Unwrap:* `Assess(case.children.0)`,
		*Unwrap:* `Assess(case.children.1)`,
	].
	2. *For each* `it` in `data`:
		1. *Assert:* `it.count` is 2.
	3. *Return:* a new Mapping containing the pairs in the sequence `data`.
;

Boolean! Assess(SemanticOperation[operator: EMP] expr) :=
	1. *Assert:* `expr.children.count` is 1.
	2. *Let* `operand` be *Unwrap:* `Assess(expr.children.0)`.
	3. *If* `*UnwrapAffirm:* `ToBoolean(operand)`` is `false`:
		1. *Return:* `true`.
	4. *If* `operand` is of type `Integer` *and* `operand` is `0`:
		1. *Return:* `true`.
	5. *If* `operand` is of type `Float` *and* `operand` is `0.0` or `-0.0`:
		1. *Return:* `true`.
	6. *If* `operand` is an instance of `String` *and* `operand` contains 0 codepoints:
		1. *Return:* `true`.
+	7. *If* `operand` is an instance of `Tuple` or `Record` or `Mapping` *and* `operand.count` is 0:
+		1. *Return:* `true`.
	8. *Return:* `false`.
;
```

## Identity
```diff
Boolean Identical(Object a, Object b) :=
	1. *If* `a` is the value `null` and `b` is the value `null`:
		1. *Return:* `true`.
	2. *If* `a` is the value `false` *and* `b` is the value `false`:
		1. *Return:* `true`.
	3. *If* `a` is the value `true` *and* `b` is the value `true`:
		1. *Return:* `true`.
	4. *If* `a` is an instance of `Integer` *and* `b` is an instance of `Integer`:
		1. If `a` and `b` have the same bitwise encoding:
			1. *Return:* `true`.
	5. *If* `a` is an instance of `Float` *and* `b` is an instance of `Float`:
		1. If `a` and `b` have the same bitwise encoding:
			1. *Return:* `true`.
+	6. *If* `a` and `b` are the same object:
+		1. *Return:* `true`.
	7. Return `false`.
;
```

## Equality
```diff
Boolean Equal(Object a, Object b) :=
	1. *If* `Identical(a, b)` is `true`:
		1. *Return:* `true`.
	2. *If* `a` is an instance of `Integer` *or* `b` is an instance of `Integer`:
		1. *If* `a` is an instance of `Float` *or* `b` is an instance of`Float`:
			1. *Return:* `Equal(Float(a), Float(b))`.
	3. *If* `a` is an instance of `Float` *and* `b` is an instance of `Float`:
		1. *If* `a` is `0.0` *and* `b` is `-0.0`:
			1. *Return:* `true`.
		2. *If* `a` is `-0.0` *and* `b` is `0.0`:
			1. *Return:* `true`.
+	4. *If* `a` is an instance of `Tuple` *and* `b` is an instance of `Tuple`:
+		1. *Let* `seq_a` be a new Sequence whose items are exactly the items in `a`.
+		2. *Let* `seq_b` be a new Sequence whose items are exactly the items in `b`.
+		3. *If* `seq_a.count` is not `seq_b.count`:
+			1. *Return:* `false`.
+		4. Assume *UnwrapAffirm:* `Equal(a, b)` is `false`, and use this assumption when performing the following step.
+			1. *Note:* This assumption prevents an infinite loop,
+				if `a` and `b` ever recursively contain themselves or each other.
+		5. *For index* `i` in `seq_b`:
+			1. *If* *UnwrapAffirm*: `Equal(seq_a[i], seq_b[i])` is `false`:
+				1. *Return:* `false`.
+		6. *Return:* `true`.
+	5. *If* `a` is an instance of `Record` *and* `b` is an instance of `Record`:
+		1. *Let* `struct_a` be a new Structure whose properties are exactly the properties in `a`.
+		2. *Let* `struct_b` be a new Structure whose properties are exactly the properties in `b`.
+		3. *If* `struct_a.count` is not `struct_b.count`:
+			1. *Return:* `false`.
+		4. Assume *UnwrapAffirm:* `Equal(a, b)` is `false`, and use this assumption when performing the following step.
+			1. *Note:* This assumption prevents an infinite loop,
+				if `a` and `b` ever recursively contain themselves or each other.
+		5. *For key* `k` in `struct_b`:
+			1. *If* `struct_a[k]` is not set:
+				1. *Return:* `false`.
+			2. *If* *UnwrapAffirm*: `Equal(struct_a[k], struct_b[k])` is `false`:
+				1. *Return:* `false`.
+		6. *Return:* `true`.
+	6. *If* `a` is an instance of `Mapping` *and* `b` is an instance of `Mapping`:
+		1. *Let* `data_a` be a new Sequence of 2-tuples,
+			whose items are exactly the antecedents and consequents in `a`.
+		2. *Let* `data_b` be a new Sequence of 2-tuples,
+			whose items are exactly the antecedents and consequents in `b`.
+		3. *If* `data_a.count` is not `data_b.count`:
+			1. *Return:* `false`.
+		4. Assume *UnwrapAffirm:* `Equal(a, b)` is `false`, and use this assumption when performing the following step.
+			1. *Note:* This assumption prevents an infinite loop,
+				if `a` and `b` ever recursively contain themselves or each other.
+		5. *For each* `it_b` in `data_b`:
+			1. Find an item `it_a` in `data_a` such that *UnwrapAffirm:* `Equal(it_a.0, it_b.0)` is `true`.
+			2. *If* no such item `it_a` is found:
+				1. *Return:* `false`.
+			3. *If* *UnwrapAffirm:* `Equal(it_a.1, it_b.1)` is `false`:
+				1. *Return:* `false`.
+		6. *Return:* `true`.
	7. Return `false`.
;
```

## Subtype
```diff
Boolean Subtype(Type a, Type b) :=
	1. *If* *UnwrapAffirm:* `Identical(a, b)`:
		// 2-7 | `A <: A`
		1. *Return:* `true`.
	2. *If* *UnwrapAffirm:* `IsEmpty(a)`:
		// 1-1 | `never <: T`
		1. *Return:* `true`.
	3. *If* *UnwrapAffirm:* `IsEmpty(b)`:
		// 1-3 | `T       <: never  <->  T == never`
		1. *Return:* `IsEmpty(a)`.
	4. *If* *UnwrapAffirm:* `IsUniverse(a)`:
		// 1-4 | `unknown <: T      <->  T == unknown`
		1. *Return:* `IsUniverse(b)`.
	5. *If* *UnwrapAffirm:* `IsUniverse(b)`:
		// 1-2 | `T     <: unknown`
		1. *Return:* `true`.
	6. *If* `a` is the intersection of some types `x` and `y`:
		1. *If* *UnwrapAffirm:* `Identical(x, b)` *or* *UnwrapAffirm:* `Identical(y, b)`:
			// 3-1 | `A  & B <: A  &&  A  & B <: B`
			1. *Return:* `true`.
		2. *If* *UnwrapAffirm:* `Subtype(x, b)` *or* *UnwrapAffirm:* `Subtype(y, b)`:
			// 3-8 | `A <: C  \|\|  B <: C  -->  A  & B <: C`
			1. *Return:* `true`.
	7. *If* `b` is the intersection of some types `x` and `y`:
		1. *If* *UnwrapAffirm:* `Subtype(a, x)` *or* *UnwrapAffirm:* `Subtype(a, y)`:
			// 3-5 | `A <: C    &&  A <: D  <->  A <: C  & D`
			1. *Return:* `true`.
	8. *If* `a` is the union of some types `x` and `y`:
		1. *If* *UnwrapAffirm:* `Subtype(x, b)` *or* *UnwrapAffirm:* `Subtype(y, b)`:
			// 3-7 | `A <: C    &&  B <: C  <->  A \| B <: C`
			1. *Return:* `true`.
	9. *If* `b` is the union of some types `x` and `y`:
		1. *If* *UnwrapAffirm:* `Identical(a, x)` *or* *UnwrapAffirm:* `Identical(a, y)`:
			// 3-2 | `A <: A \| B  &&  B <: A \| B`
			1. *Return:* `true`.
		2. *If* *UnwrapAffirm:* `Subtype(a, x)` *or* *UnwrapAffirm:* `Subtype(a, y)`:
			// 3-6 | `A <: C  \|\|  A <: D  -->  A <: C \| D`
			1. *Return:* `true`.
+	10. *If* `Equal(a, Tuple)` *and* `Equal(b, Tuple)`:
+		1. *Let* `seq_a` be a Sequence whose items are exactly the items in `a`.
+		2. *Let* `seq_b` be a Sequence whose items are exactly the items in `b`.
+		3. *If* `seq_a.count` is less than `seq_b.count`:
+			1. *Return:* `false`.
+		4. *For index* `i` in `seq_b`:
+			1. *If* *UnwrapAffirm:* `Subtype(seq_a[i], seq_b[i])` is `false`:
+				1. *Return:* `false`.
+		5. *Return:* `true`.
+	11. *If* `Equal(a, Record)` *and* `Equal(b, Record)`:
+		1. *Let* `struct_a` be a Structure whose properties are exactly the properties in `a`.
+		2. *Let* `struct_b` be a Structure whose properties are exactly the properties in `b`.
+		3. *If* `struct_a.count` is less than `struct_b.count`:
+			1. *Return:* `false`.
+		4. *For key* `k` in `struct_b`:
+			1. *If* `struct_a[k]` is not set:
+				1. *Return:* `false`.
+			2. *If* *UnwrapAffirm:* `Subtype(struct_a[k], struct_b[k])` is `false`:
+				1. *Return:* `false`.
+		5. *Return:* `true`.
+	12. *If* `Equal(a, Mapping)` *and* `Equal(b, Mapping)`:
+		1. *Let* `ak` be the union of antecedent types in `a`.
+		2. *Let* `av` be the union of consequent types in `a`.
+		3. *Let* `bk` be the union of antecedent types in `b`.
+		4. *Let* `bv` be the union of consequent types in `b`.
+		5. *If* *UnwrapAffirm:* `Subtype(ak, bk)` is `true` *and* *UnwrapAffirm:* `Subtype(av, bv)` is `true`:
+			1. *Return:* `true`.
	13. *If* every value that is assignable to `a` is also assignable to `b`:
		1. *Note:* This covers all subtypes of `Object`, e.g., `Subtype(Integer, Object)` returns true
			because an instance of `Integer` is an instance of `Object`.
		2. *Return:* `true`.
	14. *Return:* `false`.
;
```
