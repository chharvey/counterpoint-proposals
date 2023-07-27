The **type property accesss** syntax for types is analogous to the property access syntax of values.
It accesses the index or key of a tuple or record type respectively.
```cp
type T = [bool, int, str];
type T1 = T.1;             %== int
type T_1 = T.-1;           %== str
type T3 = T.3;             %> TypeError

type R = [a: bool, b?: int, c: str];
type Ra = R.a;                       %== bool
type Rc = R.b;                       %== int | void
type Rd = R.d;                       %> TypeError
```

- [x] documentation
- [x] syntax
- [x] decorate
- [x] semantics
- [x] assess for non-optional entries
- [x] assess for optional entries
