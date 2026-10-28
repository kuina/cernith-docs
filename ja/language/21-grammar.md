# 文法

Cernith の構文を、EBNF で示します。字句の規則は [02 字句](02-lexical-structure.md)、書式の規則は [19 正準形](19-canonical-form.md) にあります。

## 記法

```text
規則 = 選択肢 ;          規則の定義（1 つの名前に 1 つの規則）
a | b                    選択
[ a ]                    省略可
{ a }                    0 回以上の繰り返し
( a )                    グループ
"text"                   キーワードや記号（そのままの綴り）
大文字の名前             字句解析器が作るトークンの種類
小文字の名前             文法の非終端記号
```

トークンの種類は、`IDENT`（小文字の名前）、`TYPE_IDENT`（大文字の名前）、`INT`、`FLOAT`、`STRING` です。空白とコメントは文法には含まれません。この文脈自由文法は、字句解析器が生成するトークンを対象とし、改行やインデントには依存しません。

構文解析では、あとで別の検査によって拒否される書き方も受け付けます。ラベルのない引数、型注釈のない `var`、正準形でない書式などが該当し、それぞれ専用の診断が出ます。構文エラー（`E0001`）は、構文解析できないテキストに対して報告されます。

## ファイルと定義

```text
file            = module_header { import_decl } { item } ;

module_header   = "module" path ";" ;
import_decl     = "import" path [ "as" IDENT ] ";" ;
path            = IDENT { "." IDENT } ;

item            = type_decl | record_decl | variant_decl | const_decl | fn_decl
                | example_decl | property_decl
                | resource_decl | capability_decl | manifest_decl
                | interface_decl | impl_decl | unit_decl ;

type_decl       = [ "pub" ] "type" TYPE_IDENT [ type_params ] "=" type_rhs ";" ;
type_rhs        = "distinct" type
                | type "where" expr
                | type ;
record_decl     = [ "pub" ] "record" TYPE_IDENT [ type_params ] "{" { field_decl } "}"
                  [ "where" expr ";" ] ;
field_decl      = IDENT ":" type "," ;
variant_decl    = [ "pub" ] "variant" TYPE_IDENT [ type_params ] "{" { variant_case } "}" ;
variant_case    = TYPE_IDENT [ "(" [ field_list ] ")" ] "," ;
field_list      = field_item { "," field_item } [ "," ] ;
field_item      = IDENT ":" type ;
const_decl      = [ "pub" ] "const" IDENT ":" type "=" expr ";" ;

resource_decl   = [ "pub" ] "resource" TYPE_IDENT "release" path ";" ;
capability_decl = [ "pub" ] "capability" TYPE_IDENT [ "{" { field_decl } "}" ] ";" ;
manifest_decl   = [ "pub" ] "manifest" TYPE_IDENT "{" manifest_field { "," manifest_field } [ "," ] "}" ;
manifest_field  = IDENT ":" type_path [ "(" [ field_args ] ")" ] ;
field_args      = field_arg { "," field_arg } [ "," ] ;
field_arg       = IDENT ":" expr ;

type_params     = "<" type_param { "," type_param } ">" ;
type_param      = TYPE_IDENT [ ":" bound ] | "effect" TYPE_IDENT ;
bound           = bound_path { "+" bound_path } ;
bound_path      = { IDENT "." } TYPE_IDENT ;
```

`type` 宣言の `where` は、`self` で値を参照します。`record` の `where` は、フィールドの名前をそのまま使います（[07 レコードとバリアント](07-records-and-variants.md)）。`bound` は `Eq`、`Ord`、またはインターフェースのパスです。複数の境界は `+` でつなぎます（[22 インターフェース](22-interfaces.md)）。`type_rhs` の 3 つ目の形（`type` だけ）は型の別名で、文法上は読めますが `E0314` で拒否されます。

## 関数、example、property

```text
fn_decl         = { foreign_line } [ "pub" ] [ "spec" ] "fn" IDENT [ type_params ]
                  "(" [ param_list ] ")" "->" type [ "uses" effect_list ] fn_body ;
foreign_line    = "foreign" IDENT STRING ";" ;
param_list      = param { "," param } [ "," ] ;
param           = IDENT ":" type ;
effect_list     = effect { "," effect } ;
effect          = IDENT | TYPE_IDENT ;

fn_body         = "{" { fn_clause } block_body "}" ;
fn_clause       = "requires" expr ";" | "ensures" expr ";" | "decreases" expr ";" ;
loop_clause     = "invariant" expr ";" | "decreases" expr ";" ;

example_decl    = "example" IDENT "for" IDENT "{" expr "}" ;
property_decl   = "property" IDENT "for" IDENT "(" [ param_list ] ")"
                  "{" { property_clause } block_body "}" ;
property_clause = "requires" expr ";" ;
```

効果の名前（`io`、`clock`、`random`、`diverge`、`async`）は、`IDENT` として読まれます。効果パラメータは `TYPE_IDENT` です。`foreign` 行のバックエンド名（`python`、`javascript`、`wasm`、`native`）は `IDENT` として読まれ、これ以外の名前はあとの段階で `E0212` により拒否されます（[15 foreign 関数](15-foreign-functions.md)）。

## 型

```text
type            = fn_type | named_type | quantity_type ;
named_type      = { IDENT "." } TYPE_IDENT [ "<" type { "," type } ">" ] ;
fn_type         = "fn" "(" [ fn_type_params ] ")" "->" type [ "uses" effect_list ] ;
fn_type_params  = param { "," param } [ "," ] ;
type_path       = { IDENT "." } TYPE_IDENT ;
```

## ブロックと文

```text
block_body      = [ { stmt } simple_stmt ]
                | { stmt } tail ;
stmt            = simple_stmt | blocklike { ";" } ;
tail            = expr ;

simple_stmt     = "let" IDENT [ ":" type ] "=" expr ";"
                | "var" IDENT [ ":" type ] "=" expr ";"
                | IDENT "=" expr ";"
                | "return" expr ";"
                | "discard" expr ";"
                | "assert" expr ";"
                | call_stmt ";" ;

call_stmt       = name_base { suffix } ;

blocklike       = if_expr | match_expr | for_expr | while_expr | using_expr ;

if_expr         = "if" expr plain_block [ "else" ( if_expr | plain_block ) ] ;
plain_block     = "{" block_body "}" ;
match_expr      = "match" expr "{" { arm } "}" ;
arm             = or_pattern "=>" ( expr | plain_block ) "," ;
for_expr        = "for" pattern "in" iterable "{" { loop_clause } block_body "}" ;
while_expr      = "while" expr "{" { loop_clause } block_body "}" ;
using_expr      = "using" IDENT "=" expr plain_block ;
iterable        = range | expr ;
range           = add_expr ( ".." | "..=" ) add_expr ;
```

ブロックは、文の並びと、省略できる末尾の式（ブロックの値）でできています。単純な文は `;` で終わります。ブロックを持つ文（`if`、`match`、`for`、`while`、`using`）は `;` を付けず、ブロックの最後の要素になれば、それが末尾の式です。ブロックを持つ文のあとの余分な `;` は読み捨てられます。`let` の型注釈は文法上は読め、`E0323` で拒否されます。`for` の見出しの `pattern` は、文法では任意のパターンですが、チェッカーは名前と `_` だけを受け付けます。

## パターン

```text
pattern         = "_" | IDENT | literal_pattern | ctor_pattern ;
literal_pattern = [ "-" ] INT | STRING | "true" | "false" ;
ctor_pattern    = ctor_path [ "(" [ pat_fields ] ")" ] ;
ctor_path       = { IDENT "." } TYPE_IDENT [ "." TYPE_IDENT ] ;
pat_fields      = pat_field { "," pat_field } [ "," ] ;
pat_field       = IDENT ":" or_pattern ;
or_pattern      = pattern { "|" pattern } ;
```

## 式

優先順位の低いものから順に並べます。

```text
expr            = quantifier | blocklike | implies_expr ;
quantifier      = ( "forall" | "exists" ) IDENT ( "in" iterable | ":" type ) ":" expr ;
implies_expr    = or_expr [ "implies" implies_expr ] ;
or_expr         = and_expr { "or" and_expr } ;
and_expr        = not_expr { "and" not_expr } ;
not_expr        = "not" not_expr | cmp_expr ;
cmp_expr        = concat_expr [ cmp_op concat_expr ] ;
cmp_op          = "==" | "!=" | "<" | "<=" | ">" | ">=" ;
concat_expr     = add_expr { "++" add_expr } ;
add_expr        = mul_expr { ( "+" | "-" ) mul_expr } ;
mul_expr        = unary_expr { ( "*" | "/" ) unary_expr } ;
unary_expr      = "-" unary_expr | await_expr | postfix_expr ;
await_expr      = "await" postfix_expr_no_try { "?" } ;
postfix_expr    = base { suffix } ;
postfix_expr_no_try = base { suffix_no_try } ;
base            = name_base | other_base ;
name_base       = IDENT | TYPE_IDENT ;
other_base      = INT | FLOAT | STRING | "true" | "false" | "self"
                | "(" ")" | "(" expr ")" | list_lit | fn_lit ;
suffix          = "." IDENT | "." TYPE_IDENT
                | [ "::" "<" type { "," type } ">" ] "(" [ arg_list ] ")"
                | "[" expr "]"
                | "?" ;
suffix_no_try   = "(" [ arg_list ] ")" | "." IDENT | "[" expr "]"
                | "::" "<" type { "," type } ">" "(" [ arg_list ] ")" ;
arg_list        = arg { "," arg } [ "," ] ;
arg             = [ IDENT ":" ] expr ;
list_lit        = "[" [ expr { "," expr } [ "," ] ] "]" ;
fn_lit          = "fn" "(" [ param_list ] ")" "->" type [ "uses" effect_list ] fn_body ;
```

- `if`、`match`、`for`、`while`、`using` は、式の先頭にだけ書けます。演算子のオペランドや、後ろに `.name`・`[i]` を続ける基の式としては、括弧なしに書けません。この制約が、`blocklike` が `expr` の選択肢にだけ現れ、`unary_expr` 以下に現れない理由です。
- `arg` の `IDENT ":"` は、文法上は省略できますが、省略するとエラー `E0304` です（添字は別の記法 `"[" expr "]"` です）。
- `hole(reason: "説明")` は、予約された組み込みの名前 `hole` の普通の呼び出しです。専用の文法はありません。
- 範囲 `a..b`、`a..=b` は、`for` と量化子の中だけに現れます。

## 優先順位の表

優先順位が高い順に並べています。

| 強さ | 演算子 | 結合 |
| --- | --- | --- |
| 1（強い） | 後置：`.name`、`(...)`、`[i]`、`?` | 左 |
| 2 | 単項 `-`、`await` | |
| 3 | `*`、`/` | 左 |
| 4 | `+`、`-` | 左 |
| 5 | `++` | 左 |
| 6 | `==`、`!=`、`<`、`<=`、`>`、`>=` | 連鎖不可 |
| 7 | `not` | 前置 |
| 8 | `and` | 左 |
| 9 | `or` | 左 |
| 10（弱い） | `implies` | 右 |
| 11（最も弱い） | `forall`、`exists`、ブロックを持つ式 | |

`await` のオペランドの直後に書いた `?` は `await` に付きます。`await e?` は `(await e)?` です（[18 並行処理](18-concurrency.md)）。

## インターフェースと単位

```text
interface_decl  = [ "pub" ] "interface" TYPE_IDENT "{" { interface_member } "}" ;
interface_member = method_sig | law_decl ;
method_sig      = "fn" IDENT "(" [ param_list ] ")" "->" type [ "uses" effect_list ] ";" ;
law_decl        = "law" IDENT "(" [ param_list ] ")" "{" { "requires" expr ";" } block_body "}" ;

impl_decl       = "impl" [ type_params ] type_path "for" type
                  ( "derive" ";" | "{" { fn_decl } "}" ) ;

unit_decl       = [ "pub" ] "unit" IDENT ";" ;
quantity_type   = ( "Int" | "Float64" ) "<" unit_expr ">" ;
unit_expr       = ( unit_product | "1" ) [ "/" unit_den ] ;
unit_product    = unit_factor { "*" unit_factor } ;
unit_den        = unit_factor | "(" unit_product ")" ;
unit_factor     = { IDENT "." } IDENT [ "^" INT ] ;
```

- `interface`、`impl`、`law`、`derive` は予約語ではありません。上の文法規則で指定された位置でのみキーワードとして扱われるため、通常の名前としても引き続き使えます。`unit` は予約語です。
- or パターンのすべての選択肢が同じ名前を束縛すること（`E0350`）と、ケースまたはリテラルを宣言順に並べること（`E0351`）、インターフェースが最初の版の規則に従うこと（`E1401`）、単位が正準形で書かれること（`E0362`）は、文法ではなくチェッカーが要求します。`for` の頭は `or_pattern` ではなく `pattern` を 1 つ取ります。[22 インターフェース](22-interfaces.md)と[23 単位](23-units-of-measure.md)を参照してください。

## 関連

- 字句：[02 字句](02-lexical-structure.md)
- 式：[04 式](04-expressions.md)
