# Grammar

The syntax of Cernith is given in EBNF. The lexical rules are in [02 Lexical structure](02-lexical-structure.md), and the formatting rules are in [19 Canonical form](19-canonical-form.md).

## Notation

```text
rule = alternatives ;    definition of a rule (one rule per name)
a | b                    choice
[ a ]                    optional
{ a }                    repetition, zero or more times
( a )                    grouping
"text"                   keyword or symbol (spelled exactly as written)
uppercase name           kind of token produced by the lexer
lowercase name           nonterminal of the grammar
```

The token kinds are `IDENT` (a lowercase name), `TYPE_IDENT` (an uppercase name), `INT`, `FLOAT`, and `STRING`. Whitespace and comments do not appear in the grammar. The grammar is a context-free grammar over the tokens produced by the lexer, and it does not depend on layout.

The grammar reads a wider language than canonical form accepts. Arguments without labels, a `var` without a type annotation, non-canonical formatting, and so on can be read grammatically, and later stages reject them with individual diagnostics. The syntax error (`E0001`) is for text that cannot be read at all.

## Files and definitions

```text
file            = module_header { import_decl } { item } ;

module_header   = "module" path ";" ;
import_decl     = "import" path [ "as" IDENT ] ";" ;
path            = IDENT { "." IDENT } ;

item            = type_decl | record_decl | variant_decl | const_decl | fn_decl
                | example_decl | property_decl
                | resource_decl | capability_decl | manifest_decl ;

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
type_param      = TYPE_IDENT [ ":" TYPE_IDENT ] | "effect" TYPE_IDENT ;
```

The `where` of a `type` declaration refers to the value with `self`; the `where` of a `record` uses the names of its fields as they are ([07 Records and variants](07-records-and-variants.md)). The `TYPE_IDENT : TYPE_IDENT` in `type_param` is a bound (`Eq`, `Ord`). The third form of `type_rhs` (just `type`) is a type alias; it can be read grammatically but is rejected with `E0314`.

## Functions, examples, and properties

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

The names of effects (`io`, `clock`, `random`, `diverge`, `async`) are read as `IDENT`. Effect parameters are `TYPE_IDENT`. The backend name in a `foreign` line (`python`, `javascript`, `wasm`, `native`) is read as an `IDENT`; any other name is rejected later with `E0212` ([15 Foreign functions](15-foreign-functions.md)).

## Types

```text
type            = fn_type | named_type ;
named_type      = { IDENT "." } TYPE_IDENT [ "<" type { "," type } ">" ] ;
fn_type         = "fn" "(" [ fn_type_params ] ")" "->" type [ "uses" effect_list ] ;
fn_type_params  = param { "," param } [ "," ] ;
type_path       = { IDENT "." } TYPE_IDENT ;
```

## Blocks and statements

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
arm             = pattern "=>" ( expr | plain_block ) "," ;
for_expr        = "for" pattern "in" iterable "{" { loop_clause } block_body "}" ;
while_expr      = "while" expr "{" { loop_clause } block_body "}" ;
using_expr      = "using" IDENT "=" expr plain_block ;
iterable        = range | expr ;
range           = add_expr ( ".." | "..=" ) add_expr ;
```

A block consists of a sequence of statements and an optional final expression (the value of the block). Simple statements end with `;`. Statements that have blocks (`if`, `match`, `for`, `while`, `using`) take no `;`, and if one of them is the last element of a block, it is the final expression. Extra `;` after a statement that has a block are discarded. A type annotation on `let` can be read grammatically and is rejected with `E0323`. The `pattern` in the heading of a `for` is any pattern as far as the grammar is concerned, but the checker accepts only names and `_`.

## Patterns

```text
pattern         = "_" | IDENT | literal_pattern | ctor_pattern ;
literal_pattern = [ "-" ] INT | STRING | "true" | "false" ;
ctor_pattern    = ctor_path [ "(" [ pat_fields ] ")" ] ;
ctor_path       = { IDENT "." } TYPE_IDENT [ "." TYPE_IDENT ] ;
pat_fields      = pat_field { "," pat_field } [ "," ] ;
pat_field       = IDENT ":" pattern ;
```

## Expressions

They are listed from the lowest precedence to the highest.

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

- `if`, `match`, `for`, `while`, and `using` can be written only at the start of an expression. Without parentheses, they cannot be written as the operand of an operator, or as the base expression followed by `.name` or `[i]`. This restriction is why `blocklike` appears only as an alternative of `expr` and not at or below `unary_expr`.
- The `IDENT ":"` in `arg` can be omitted as far as the grammar is concerned, but omitting it is the error `E0304` (indexing uses a different notation, `"[" expr "]"`).
- `hole(reason: "explanation")` is an ordinary call to the reserved built-in name `hole`. There is no dedicated grammar for it.
- The ranges `a..b` and `a..=b` appear only in `for` and in quantifiers.

## Precedence table

From the strongest binding to the weakest.

| Strength | Operators | Associativity |
| --- | --- | --- |
| 1 (strongest) | Postfix: `.name`, `(...)`, `[i]`, `?` | Left |
| 2 | Unary `-`, `await` | |
| 3 | `*`, `/` | Left |
| 4 | `+`, `-` | Left |
| 5 | `++` | Left |
| 6 | `==`, `!=`, `<`, `<=`, `>`, `>=` | Cannot be chained |
| 7 | `not` | Prefix |
| 8 | `and` | Left |
| 9 | `or` | Left |
| 10 (weak) | `implies` | Right |
| 11 (weakest) | `forall`, `exists`, expressions that have blocks | |

`?` written directly after the operand of `await` belongs to the `await`: `await e?` is `(await e)?` ([18 Concurrency](18-concurrency.md)).

## Related

- Lexical structure: [02 Lexical structure](02-lexical-structure.md)
- Expressions: [04 Expressions](04-expressions.md)
