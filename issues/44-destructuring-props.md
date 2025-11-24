Just like with variables (#43), we can declare properties of a record literal using destructuring syntax.

# Discussion
Rather than assigning properties of a record individually, we can use destructuring syntax:
```cp
let my_data: [int, int, int, int] = [42, 420, 4200, 42000];
let my_record: [x: int, y: int] = [[x, y]= my_data];
my_record.x; %== 42
my_record.y; %== 420
```
The syntax `[[x, y]= my_data]` is shorthand for `[x= my_data.0, y= my_data.1]`.

In the example above, we used tuple destructuring, but we could have just as well used record destructuring. Both are well-formed inside a record literal.
```cp
let my_data: [x: int, y: int, z: int] = [x= 42, y= 420, z= 4200];
let my_record: [x: int, y: int] = [[$y, $x]= my_data];
my_record.x; %== 42
my_record.y; %== 420
```
`[[$y, $x]= my_data]` is shorthand for `[y= my_data.y, x= my_data.x]`.

As with all record destructuring, we can use aliases instead of `$` punning:
```cp
let my_data: [xray: int, yankee: int, zulu: int] = [xray= 42, yankee= 420, zulu= 4200];
let my_record: [x: int, y: int] = [[yankee= y, xray= x]= my_data];
my_record.x; %== 42
my_record.y; %== 420
```
`[[yankee= y, xray= x]= my_data]` is shorthand for `[y= my_data.yankee, x= my_data.xray]`.

It’s worth mentioning that destructuring inside a record literal is a bit like spreading (#64).
```cp
let font_styles: FontStyles = [
	weight= 600,
	style=  "oblique",
	size=   16,
];
let all_styles: Styles = [
	color=     "#0000ff",
	underline= true,
	[$weight, $style, $size]= font_styles, % kind of like spreading
];
all_styles == [
	color=     "#0000ff",
	underline= true,
	weight=    600,
	style=     "oblique",
	size=      16,
]; %== true
```
In contrast with spreading, we must explicitly list out all of the record’s properties when destructuring, and any properties that we leave out are not entered into the record.

But we can destructure a tuple inside a record, which is not possible with spread.
```cp
let all_styles: Styles = [
	color=     "#0000ff",
	underline= true,
	[weight, style, size]= [600, "oblique", 16], % impossible with spreading
];
```

Type errors are raised when the assigned expression of a destructuring statement doesn’t match the assignee.
```cp
let my_dict: [:int] = [
	[a, b, c]= [42, 420, 4200, 42000],                % `42000` is dropped, but no error
	[d, e, f]= [42, 420],                             %> TypeError (index `2` is missing)
	[$a, $b, $c]= [a= 42, b= 420, c= 4200, d= 42000], % `d` is dropped, but no error
	[$d, $e, $f]= [d= 42, e= 420],                    %> TypeError (property `f` is missing)
	[golf= g, hotel= h]= [golf= 42, h= 420],          %> TypeError (property `hotel` is missing)
];
let my_record: [a: int, b: int, c: int, d: int, e: int] = [
	[a, b, c]= [42, 420, 123.45],          %> TypeError (`123.45` is not an int)
	[$d, echo= e]= [d= null, echo= "420"], %> TypeError
];
```

Destructuring for property declaration can be recursive: We can nest destructured patterns within each other.
```cp
let rec = [
	% regular property destructuring, tuple
	[a, b]= [1, 2],

	% regular property destructuring, record
	[$c, delta= d]= [c= 3, delta= 4],

	% nested property destructuring, tuple within tuple
	[g, [h, i]]= [7, [8, 9]],

	% nested property destructuring, record within tuple
	[j, [$k, lima= l]]= [10, [k= 11, lima= 12]],

	% nested property destructuring, tuple within record
	[$m, november= [n, o]]= [m= 13, november= [14, 15]],

	% nested property destructuring, record within record
	[papa= p, quebec= [$q, romeo= r]]= [papa= 16, quebec= [q= 17, romeo= 18]],
];

rec == [
	a= 1,  b= 2,  c= 3,  d= 4,  g= 7,  h= 8,  i= 9,  j= 10,
	k= 11, l= 12, m= 13, n= 14, o= 15, p= 16, q= 17, r= 18,
]; %== true
```

# Specfication
## Syntax Grammar
```diff
+DestructurePropertyItem
+	::= Word | DestructureProperties;

+DestructurePropertyKey ::=
+	| "$" Word
+	| Word "=" DestructurePropertyItem
+;

+DestructureProperties ::=
+	| "[" ","? DestructurePropertyItem# ","? "]"
+	| "[" ","? DestructurePropertyKey#  ","? "]"
+;

Property ::=
	| "$" IDENTIFIER
-	| Word                           "=" Expression
+	| (Word | DestructureProperties) "=" Expression
;
```
