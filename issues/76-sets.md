Sets are dynamic-sized unordered lists of values.

```cp
let seasons: obj = {'winter', 'spring', 'summer', 'fall'};
```

Sets cannot contain identical (via `===`) elements. The set `{'hello', 'hello'}` only contains 1 element.

Sets may have several elements that are un-identical but “equal”.
```cp
let x: [str] = ['hello'];
let y: [str] = ['hello'];
let set: obj = {0.0, -0.0, x, y};
```
Even though `0.0 == -0.0` and `x == y`, this set has four elements.

Sets are considered equal (via `==`) iff they have equal elements. The order of elements in a set is not relevant.
```cp
let x: obj = {'hello', 'world'};
let y: obj = {'world', 'hello'};
x === y; %== false
x == y;  %== true
```

Set elements can be accessed via bracket accessor notation. If the expression in the brackets exists in the set, it is produced. (In a way, think of a Set as a Mapping whose keys and corresponding values are the same). If the compiler can determine that the value does not exist in the set, a VoidError is thrown; otherwise it’s a runtime error. An expression of an incorrect type is a TypeError.
```cp
{'hello', 'world'}.['hello']; %== 'hello'
{'hello', 'world'}.['hi'];    % compile-time VoidError or runtime error
{'hello', 'world'}.[42];      % compile-time TypeError
```

Optional access and claim access as normal.
```cp
{'hello', 'world'}?.['hello']; %== 'hello'
{'hello', 'world'}?.['hi'];    %== null
{'hello', 'world'}!.['hello']; %== 'hello'
{'hello', 'world'}!.['hi'];    %: str      % claims type is non-null string, but results in runtime error
```

- [x] set literal syntax
- [x] decorate set literals
- [x] type set literals
- [x] assess set literals
- [x] new Set type & documentation
- [x] update Subtype algorithm
- [x] update Equal algorithm
- [x] type property access
- [x] type optional & claim access
- [x] assess property access
- [x] assess optional & claim access
