# minicalc

A small calculator for lines of text, written as a package of eight modules. It reads lines from standard input and prints one result per line:

- an expression, such as `1 + 2 * 3`, prints its value;
- an assignment, such as `x = 4 * (2 + 1)`, prints `x = 12` and remembers `x`;
- `:vars` lists the variables, sorted by name;
- an empty line is ignored.

An input error does not end the session. The calculator prints a message, the input line, and a caret at the error column, then continues with the next line.

This sample combines the individual ideas introduced elsewhere in the collection into a program of about 2000 lines. All computation lives in pure functions with examples and properties; only `main` declares `uses io`.

## Run it

From the directory that contains `examples/`:

```text
cernith run examples/larger/minicalc < examples/larger/minicalc/run.stdin
cernith test examples/larger/minicalc
cernith check --verify examples/larger/minicalc
```

`run.stdin` is the input of the transcript below, and it is also what the checker script feeds to `run`.

## A session

This is the real output of the program. Each `>` line is a line of `run.stdin`; the lines under it are what the program printed for it. (The transcript was made by running the program on every prefix of `run.stdin` and keeping what each new line added.)

```text
> 1 + 2 * 3
7
> x = 4 * (2 + 1)
x = 12
> y = x - 2
y = 10
> x * y
120
> -2 ^ 2
-4
> 2 ^ 3 ^ 2
512
> 7 / 2
3
> -7 / 2
-4
> -7 % 2
1
> gcd(84, 36)
12
> max(x, min(y, 100))
12
> abs(-5)
5
>
> big = 2 ^ 100
big = 1267650600228229401496703205376
> big * big
1606938044258990275541962092341162602522202993782792835301376
> :vars
big = 1267650600228229401496703205376
x = 12
y = 10
> 10 / (x - 12)
error at column 4: division by zero
10 / (x - 12)
   ^
> 2 ^ -1
error at column 3: negative exponent
2 ^ -1
  ^
> foo + 1
error at column 1: unknown variable `foo`
foo + 1
^
> sqrt(16)
error at column 1: unknown function `sqrt`
sqrt(16)
^
> min(1)
error at column 1: `min` takes 2 arguments, got 1
min(1)
^
> abs = 3
error at column 1: cannot assign to the function `abs`
abs = 3
^
> (1 + 2
error at column 7: missing `)`
(1 + 2
      ^
> 3 $ 4
error at column 3: unexpected character `$`
3 $ 4
  ^
> 2 +
error at column 4: unexpected end of input
2 +
   ^
> 1 2
error at column 3: unexpected `2`
1 2
  ^
```

## The language

| Feature | Rule |
|---|---|
| Values | `Int` only, unbounded (`2 ^ 100` is exact). |
| Operators | `+ - * / % ^`, unary `-`, parentheses. |
| Precedence, tightest first | `^` (right associative), unary `-`, `* / %`, `+ -` (both left associative). So `-2 ^ 2` is `-4`, `2 ^ 3 ^ 2` is `2 ^ 9`, and `-2 * 3` is `(-2) * 3`. |
| `/` and `%` | Floor semantics: the quotient rounds down (`int.div_floor`) and the remainder has the sign of the divisor (`int.rem_floor`). `-7 / 2` is `-4` and `-7 % 2` is `1`, and `a == (a / b) * b + a % b` always holds. |
| `^` | The exponent must be at least 0 (`negative exponent`) and at most 1000 (`exponent too large`), which keeps `9 ^ 9 ^ 9` from running for hours. `0 ^ 0` is 1. |
| Names | Letters, digits and `_`, not starting with a digit. A variable must be assigned before it is used. |
| Functions | `abs(x)`, `min(a, b)`, `max(a, b)`, `gcd(a, b)` (`gcd` is never negative). Arguments are checked for count before they are evaluated. |
| Assignment | `name = expression`. A function name cannot be assigned. A failed assignment leaves the variable as it was. |
| Commands | `:vars`. Anything else starting with `:` is an unexpected character. |
| Errors | `error at column N: message`, the source line, a caret line. Columns count from 1 in the line as typed (a tab counts as one column and can shift the caret on a terminal). |

The messages: `unexpected character`, `unexpected end of input`, ``missing `)` ``, `unexpected` followed by a token, `unknown variable`, `unknown function`, `takes N arguments, got M`, `division by zero`, `negative exponent`, `exponent too large`, `cannot assign to the function`.

## Module map

Each module imports only the modules listed in its row. Dependencies form layers, from `token` and `ast` at the bottom to `main` at the top.

| Module | Lines | Imports (minicalc) | What it does |
|---|---|---|---|
| `minicalc.token` | 298 | - | The lexer: a line to tokens with columns, or the first bad character. |
| `minicalc.ast` | 177 | - | The syntax tree, operator facts (precedence, associativity), `show` to print a tree fully parenthesized. |
| `minicalc.parser` | 505 | `ast`, `token` | Precedence climbing: tokens to a statement, or a parse error. The longest module, mostly because of its examples. |
| `minicalc.arith` | 158 | `ast` | The six operators on `Int` with floor division and the exponent limit; errors without columns. |
| `minicalc.eval` | 293 | `arith`, `ast` | Walks a tree against the variables (`map.Map<String, Int>`), calls the built-in functions, adds columns to errors. |
| `minicalc.report` | 226 | `arith`, `eval`, `parser`, `token` | Words for every error, and the three-line report with the caret. |
| `minicalc.session` | 283 | `ast`, `eval`, `parser`, `report`, `token` | Runs a list of lines and returns the output lines. Pure. |
| `minicalc.main` | 93 | `session` | The only module with `uses io`: read lines, print what the session returns. |

## What to read first

1. `main.cnth`: the small entry point that handles input and output, and end-to-end examples using `test.with_world`.
2. `session.cnth`: the path from an input line through lexing, parsing, and evaluation, with errors from each stage reported consistently. The properties at the end exercise the calculator with numbers as text and compare the results with integer calculations.
3. `parser.cnth`, the header comment first: how a recursive parser proves that it terminates.
4. `ast.cnth` and `eval.cnth` for the structural recursion over the tree (no `decreases` needed).
5. `arith.cnth` for contracts that the verifier proves.

## Things to notice

- **Termination of the parser.** `parse_expr`, `parse_rest` and `parse_arguments` call each other and themselves. Each has `decreases remaining(...)`, the number of unread tokens, and `parse_arguments` adds a second `decreases` clause to break a tie (lexicographic order). A later call starts where an earlier call ended, so the checker must know that the earlier call consumed a token. That fact is stated by the `where used >= 1` clause of the result records, not by an `ensures`, because a restriction of a type holds for every value of it.
- **Structural recursion in the evaluator.** A function call node has the fixed shapes `Call1`, `Call2` and `BadCall` instead of a list of arguments. `evaluate` recurses on the parts of the node it matched, which needs no `decreases`. A recursion that walked a list of children would need a measure of its own.
- **Errors are values at every stage**, each with its own variant (`LexError`, `ParseError`, `ArithError`, `EvalError`). The report module is the one place that turns them into text. No stage can stop the program: dividing by zero is checked before `int.div_floor`, whose `requires divisor != 0` would make it a defect.
- **Contracts the verifier proves.** `arith.apply` states that `+`, `-`, `*` are exact and that a zero divisor is an error; `report.render` states the shape of the three lines; the lexer helpers state that every scan makes progress, which is what the lexer's loop needs for its `decreases`. `cernith check --verify --format json` lists the level reached by every function.
- **Verifier limits met on the way.** A recursive `spec fn` and `int.pow` (whose contract uses one) are not encoded yet, so `arith.raise` keeps the call to `int.pow` in a one-line function of its own and the contract of `apply` stays proved.
- **Properties** drive the real pipeline with generated numbers: `sums_and_products_match_the_integers`, `multiplication_binds_tighter_than_addition`, `division_rounds_down_and_remainder_follows`, `gcd_is_symmetric` (session); `lexing_loses_only_white_space`, `token_columns_point_into_the_source` (token); `the_caret_is_under_the_column` (report); `operators_match_the_integer_operators` and `powers_add_their_exponents` (arith).
