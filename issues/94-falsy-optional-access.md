## Problem Statement

A common pattern is using the `&&` operator to check if a value is truthy before accessing an property on it. E.g. `if result && result.message then { do_something.(); };`. The `result` base must be duplicated, and if it isn’t a variable (e.g. it’s some function call) it would require duplicate evaluation. We want a way of expressing `result && result.message` concisely, which would produce `result.message` if it exists, else produce `result`.

## Possible Workarounds
No workarounds. `result && result.message` is the only way to achieve this.

## Proposed Solution
The optional access operator should be modified for this behavior. Currently, `result?.message` returns `result.message` if it exists, else `null`. However, if `result` is not `null` but another falsy value, such as the result of a void function call, the boolean value `false`, or an Exception object, the expression `result?.message` is a type-error. `result?.message` should return `result.message` if it exists, else `result && null`.

### Example
```cp
% before:
claim nullish_obj: null | [prop?: P];
nullish_obj?.prop; %: null | P

claim falsy_obj: false | [prop?: P];
falsy_obj?.prop; %> TypeError: `prop` is not a key of `false`

% after:
claim nullish_obj: null | [prop?: P];
nullish_obj?.prop; %: null | P

claim falsy_obj: false | [prop?: P];
falsy_obj?.prop; %: false | null | P
%^ produces `false` if `falsy_obj == false`,
%  produces `null` if `falsy_obj` is a record with no `prop` key,
%  produces a `P` value if `falsy_obj` is a record with a `prop` key
```

### Compatibility
Is this feature a “breaking” change? In other words, if this feature is introduced, will users’ existing code break and need to be changed/updated as a result? (Select only one.)
- ( ) yes, this feature is “breaking”
- (x) no, this feature is “non-breaking”
- ( ) not sure

### Benefits
More concise code, less duplicate evaluation. Current optional access expressions `a?.b` don’t need to change. Current expressions like `a && a.b` don’t need to change but may update at-will.

### Drawbacks
More work for the compiler. Cases like `fn.() && fn.().message` when `fn` could return void must remember to use `fn?.()?.message` instead of `fn.()?.message`.

### Details
```diff
Type! TypeOfUnfolded(SemanticAccess access) :=
# ...
	5. *If* `access.kind` is `OPTIONAL`:
+		1. *Let* `falsetype` be *UnwrapAffirm:* `ToType(false)`.
+		2. *Let `falsytypes` be *UnwrapAffirm:* `Union(Void, Null, falsetype)`.
-		3. *If* *UnwrapAffirm:* `Subtype(base_type, Null)`:
+		3. *If* *UnwrapAffirm:* `Subtype(base_type, falsytypes)`:
			1. *Return:* `base_type`.
-		4. *If* *UnwrapAffirm:* `Subtype(Null, base_type)`:
+		4. *If* *UnwrapAffirm:* `Subtype(falsytypes, base_type)`:
-			1. *Set* `is_nullish` to `true`.
-			2. *Set* `base_type` to `Difference(base_type, Null)`.
+			1. *Set* `maybe_falsy` to `true`.
+			2. *Set* `base_type` to `Difference(base_type, falsytypes)`.
+			3. *Set* `base_type_falsy` to `Intersect(base_type, falsytypes)`.
# ...
	10. *If* `returned` is not *none*:
-		1. *If* `is_nullish` is `true`:
+		1. *If* `maybe_falsy` is `true`:
-			1. *Return:* `Union(returned, Null)`.
+			1. *Return:* `Union(returned, base_type_falsy)`.
		2. *Return*: `returned`.
# ...
;

Object! ValueOf(SemanticAccess access) :=
# ...
-	5. *If* `access.kind` is `OPTIONAL` *and* *UnwrapAffirm:* `Equal(base_value, null)`:
+	5. *If* `access.kind` is `OPTIONAL` *and* *UnwrapAffirm:* `IsFalsy(base_value)`:
		1. *Return:* `base_value`.
# ...
;

+Boolean IsFalsy(Object value) :=
+	1. *If* *UnwrapAffirm:* `Identical(value, null)` *or* *UnwrapAffirm:* `Identical(value, false)`:
+		1. *Return:* `true`.
+	2. *Note:* TODO: if instance of `Exception`, return `true`.
+	3. *Else:*
+		1. *Return:* `false`.
+;

# expressions-operators.md#optional-access
-If the *binding object is `null`*, then the optional access operator also produces `null`.
-For example, `null.property` is a type error (and if the compiler were bypassed,
-it would cause a runtime error), but `null?.property` will simply produce `null`.
-This facet makes optional access safe to use when chained.
+If the *binding object is falsy*, then the optional access operator also produces that value.
+For example, if `ex` is an exception, then `ex.property` is a type error (and if the compiler were bypassed,
+it would cause a runtime error), but `ex?.property` will simply produce `null`.
+This facet makes optional access safe to use when chained.
```

## Alternatives
Alternative solution would be to have a different operator, but the optional access operator can be extended and will still work for `null` values.
