The Top and Bottom types have new names: they are now called `anything` and `nothing` respectively.
The old keywords, `unknown` and `never`, are now freed for general identifiers.
```cp
type Universe = anything;
type Empty    = nothing;
%               ^ new reserved keywords

let var u: anything = 42;
set u = 4.2;
set u = null;
set u = "any value";
u.count; %> TypeError: property `count` does not exist on type `anything`.

claim e: nothing;
e.count; % type `nothing`
e.abcde; % type `nothing`
set e = 42; %> TypeError: expression of type `int` is not assignable to type `nothing`.

% old identifiers are now available
let var unknown: int = 42; % ok
type unknown = int;        % ok
let var never: int = 42;   % ok
type never = int;          % ok

let var u: unknown = 42; %> ReferenceError: `unknown` is not declared
claim e: never;          %> ReferenceError: `never` is not declared
```

## Lexicon
```diff
KeywordType :::=
-	| "never"
+	| "nothing"
	| "bool"
	| "sym"
	| "int"
	| "float"
	| "str"
-	| "unknown"
+	| "anything"
;
```

## TokenWorth
```diff
-TokenWorth(KeywordType  :::= "never")    -> RealNumber := \x80;
+TokenWorth(KeywordType  :::= "nothing")  -> RealNumber := \x80;

-TokenWorth(KeywordType  :::= "unknown")  -> RealNumber := \x86;
+TokenWorth(KeywordType  :::= "anything") -> RealNumber := \x86;
```

## Decorate
```diff
-KeywordType(KEYWORD_TYPE :::= "never")    -> Type := Never;
+KeywordType(KEYWORD_TYPE :::= "nothing")  -> Type := Nothing;
-KeywordType(KEYWORD_TYPE :::= "unknown")  -> Type := Unknown;
+KeywordType(KEYWORD_TYPE :::= "anything") -> Type := Anything;
```
