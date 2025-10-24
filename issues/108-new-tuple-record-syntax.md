Tuples and Records, both their types and their expressions, now use parenthesis delimiters. 1-Tuples must have a leading or trailing comma so as not to confuse it with grouping. The square bracket delimiters are now for Lists and Dicts, with some minor changes in their type shorthands. Set type shorthand has also been changed. The motivation behind this breaking change is to offer a literal syntax for each of the six built-in compound types: tuple, record, list, dict, set, and map.
```cp
% ---- TYPES ---- %
type NewTupleType = (int, int, int);
type SingletonTupleType = (str,); % or `(, str)` --- leading or trailing comma is required
type TupleAccess1 = NewTupleType.0 & NewTupleType.2; % access via integer literals
type TupleAccess2 = SingletonTupleType.0;

type NewRecordType = (a: int, b: float, c: str);
type RecordAccess = NewRecordType.a | NewRecordType.c; % access via key literals

type NewListType = [int]; % shorthand for `List.<int>` % previously `int[]`
type NewListType2 = [int | str]; % shorthand for `List.<int | str>` % previously `(int | str)[]`
% no syntax for list type access yet

type SameDictType = [:int]; % shorthand for `Dict.<int>` % hasn’t changed
% no syntax for dict type access yet

type NewSetType = {int}; % shorthand for `Set.<int>` % previously `int{}`
% no syntax for set type access yet

type SameMapType = {int -> str}; % shorthand for `Map.<int, str>` % hasn’t changed
% no syntax for map type access yet

% ---- EXPRESSIONS ---- %
let new_tuple_value: (int, float, str) = (42, 2.718, "hi");
let singleton_tuple_value: (int,) = (42,); % or `(, int)` and `(, 42)` depending on your style
let tuple_access: int | str = new_tuple_value.0 || new_tuple_value.2; % access via integer literals

let new_record_value: (a: int, b: float, c: str) = (a= 42, b= 2.718, c= "hi");
let record_access: int | str = new_record_value.a || new_record_value.c; % access via key literals

let new_list_value: [int] = [42, 43, 44]; % shorthand for `List.<int>(...)` % used to be tuple syntax
let list_access: int = new_list_value.[0] && new_list_value.[1]; % access via `int`-type expressions

let new_dict_value: [:int] = [a= 42, b= 43, c= 44]; % shorthand for `Dict.<int>(...)` % used to be record syntax
let dict_access: int = new_dict_value.[@a] && new_dict_value.[@b]; % access via `sym`-type (symbol) expressions

let same_set_value: {int} = {42, 43, 44}; % hasn’t changed
let set_access: bool = same_set_value.[42]; % access via `anything`-type expressions

let same_map_value: {int -> str} = {42 -> "hello", 69 -> "world"}; % hasn’t changed
let map_access: str = same_map_value.[42]; % access via `anything`-type expressions
```
There is no more shorthand for “tuple of 3 ints” (which used to be `int[3]`). This may be revisited, possibly updating to something like `(int * 3)` but it needs to be more fleshed out. For now, tuple types with repeated entries must be written out longhand.
