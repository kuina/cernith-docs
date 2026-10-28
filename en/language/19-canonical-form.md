# Canonical form

A program's **canonical form** is the output of the formatter (`cernith fmt`). It depends only on the syntax tree and comments, and specifies a single format for whitespace, line breaks, declaration order, and parentheses.

The grammar is independent of formatting ([02 Lexical structure](02-lexical-structure.md)), so code can be parsed even when it is not in canonical form. Non-canonical text produces a diagnostic.

- In `cernith check`, it is the warning `W0102` (non-canonical-text). The canonical text is attached, and a safe fix, “replace the file with the canonical form,” is proposed. This treatment keeps other diagnostics from being hidden or delayed.
- In a release build, if the source text does not match the canonical form, it is the error `E0101` (non-canonical-form).

All commands that emit source code, including `cernith fmt`, editing commands, and fix application, use canonical form.

The formatter has the following properties.

- **Idempotent**: formatting canonical form changes nothing.
- **Preserves meaning**: reading the result of formatting back gives the same syntax tree (except for the order of declarations, which is part of the canonical form).
- Comment strings are not changed.

Things whose order carries meaning are not reordered by formatting: record fields, variant cases, function parameters, clauses of the same kind (a conjunction, but kept in the order written), call arguments (labels must be in the declared order), and statements.

## Rules

### Text

1. UTF-8 without a BOM, LF line endings, no tabs, no trailing whitespace. The file ends with exactly one newline. No blank lines at the start or the end.
2. One level of indentation is two spaces. Opening a curly brace makes the indentation one level deeper, and the continuation lines of a wrapped parenthesized group or of a chain of operators are one level deeper.
3. Lines of code are at most 80 columns (counted in Unicode scalar values), except comment lines and lines with no place to wrap (such as a single long string literal).

### File structure

4. Comments before the `module` line go at the top, with no blank line between them and `module`.
5. The order is: the `module` line, a blank line, the `import`s (sorted by path, then by alias, with no blank lines between them), a blank line, and the definitions. A file without `import` has one blank line after `module`.
6. Definitions form three groups, in this order. Definitions are separated by one blank line.
   1. **Types**: `type`, `record`, `variant`, `resource`, `capability`, `manifest`, `interface`, `unit`. In name order, regardless of kind. The `impl` declarations come after them, in this group, sorted by the text of the interface and then of the type.
   2. **Constants**. In name order.
   3. **Functions**: all ordinary `fn`, `spec fn`, and foreign functions. In name order. Immediately after each function go its `example` declarations (in name order), then its `property` declarations (in name order).

   Names are compared in code point order (ASCII).
7. `foreign` lines go immediately before the corresponding `fn`, in order of backend name.

### Blocks and statements

8. `{` goes at the end of a line. A non-empty block has one element per line, written one level deeper, and `}` goes on its own line (except for `} else {` and `} else if condition {`). An empty block is `{}`.
9. A block begins with clauses (for functions: `requires`, `ensures`, `decreases`; for loops: `invariant`, `decreases`; for a property: `requires`). Clauses end with `;`. Exactly one blank line goes between the clauses and the other content, and no other blank lines go inside a block.
10. `let`, `var`, assignments, `return`, `discard`, `assert`, and call statements end with `;`. `if`, `match`, `for`, and `while` take no `;`, and neither does the final expression of a block. `if`, `match`, `for`, and `while` are always written in the multi-line form.
11. `match` cases are written one per line, with a trailing comma. A case with a block also ends with `},`.

### Expressions

12. One space goes after `,` and `:`, and before and after binary operators, `=`, `=>`, and `->`. No space goes inside parentheses, before `,` `:` `;`, around `.` `..` `..=` `::`, or after unary `-`. One space goes after `not`. Type arguments have no inner spaces (`List<Int>`).
13. **Parentheses** are added only when needed: when precedence or associativity requires them, and for an `and` expression that is an operand of `or`. Unneeded parentheses are removed, and a right operand of the same precedence is parenthesized (`a - (b - c)`). Parentheses on the left of a left-associative chain are merged into the chain (`(a + b) + c` becomes `a + b + c`).
14. **Wrapping** is decided starting from the outermost group. The comma-separated elements inside `( )` and `[ ]`, and chains of operators of the same precedence, form groups.
    - If the stretch from the start of the group to the next position where the outer group can wrap fits within 80 columns, and contains nothing that necessarily forces a line break (a multi-line block), the group is written on **one line**.
    - Otherwise it is **wrapped**. A parenthesized group is written with each element on its own line, one level deeper, with a comma after the last element and the closing parenthesis on its own line. A chain of operators is wrapped before each operator, and the continuation lines begin with the operator and are one level deeper.
    - Inner groups are decided independently, after the outer group has been decided.
    - A group written on one line has no trailing comma. Trailing commas are used only for the groups of call arguments, list literals, parameter lists, pattern fields, and variant case fields.
15. A function header is written on one line if it fits, including the `{`. If it does not fit, the parameter list is wrapped, and `) -> type uses effects {` goes on the line of the closing parenthesis. The effects in `uses` are listed once each, without duplicates, in lexicographic order (lowercase effect names first, then effect parameters).

### Literals

16. Integers are decimal. Booleans are `true` and `false`.
17. Floating-point numbers use the shortest decimal digit string that reads back to the same value. When the decimal exponent is from -6 to 20, positional notation is used (`123.45`, `0.000001`; at least one digit after the decimal point). Otherwise scientific notation of the form `d.ddde` followed by the exponent is used (`1.0e21`, `1.0e-7`; the exponent has a sign only when it is negative).
18. Strings are read for their content and rewritten with minimal escapes: `\\`, `\"`, `\n`, `\t`, `\r`, and `\u{hex digits}` (lowercase, no zero padding) for characters whose general category is Cc, Cf, Co, Zl, or Zp (according to the fixed table of Unicode 18.0.0). All other characters are written as they are.

### Comments

19. Comments are `//` line comments. The characters after `//`, including trailing spaces, are kept as they are. A comment after code moves to the line before the element that follows.
20. A comment belongs to the element that follows it (a definition, an `import`, a statement, a clause, a field, a case, or a `match` case) and moves together with it, at its indentation. A comment before a closing bracket stays at the end of the block, and a comment after the last definition stays at the end of the file. Comments in other positions (in the middle of an expression or of a parameter list) are errors. Comments have no meaning.

### Interfaces and patterns

21. An `interface` is laid out like a record: one member per line, indented. Method signatures follow one another without blank lines, and a blank line separates a `law` from the member before and after it. A signature that does not fit is broken like a function header. A `law` is laid out like a `property`.
22. An `impl` with methods has its functions indented, one block each, separated by blank lines. An `impl` that derives is one line (`impl show.Show for Point derive;`). An `impl` with no methods is `{}`.
23. A bound list is written `A + B`, one space on each side of `+`, in the order written: the order is the order of the instance parameters, so it is part of the meaning.
24. The alternatives of an or-pattern are joined by ` | ` in the order written. The formatter does not reorder them: only the checker knows the declaration order of the cases, and it rejects another order (`E0351`).

## Example

The following program is grammatically valid (only the warning `W0102` is reported), but it is not in canonical form.

```cernith-error W0102
module guide.messy;
import std.list;
pub fn zeta(x:Int)->Int{x+1}
pub const limit:Int=3;
example z for zeta{zeta(x:1)==2}
pub type B=distinct Int;
pub fn alpha(items:List<Int>,count:Int,other_long_parameter_name:Int,more:Int)->List<Int>{
  requires count>=0 and count<=items.length;
  ensures result.length==count;
  list.take(items:items,count:count)
}
pub record A{x:Int,y:Int,}
```

The canonical form is as follows. Blank lines are inserted after `import` and between definitions, the definitions are arranged as types, constants, and functions, the functions are in name order (`alpha`, `zeta`), and the `example` is placed immediately after `zeta`. The long header has its parameters wrapped, one per line.

```cernith
module guide.messy;

import std.list;

pub record A {
  x: Int,
  y: Int,
}

pub type B = distinct Int;

pub const limit: Int = 3;

pub fn alpha(
  items: List<Int>,
  count: Int,
  other_long_parameter_name: Int,
  more: Int,
) -> List<Int> {
  requires count >= 0 and count <= items.length;
  ensures result.length == count;

  list.take(items: items, count: count)
}

pub fn zeta(x: Int) -> Int {
  x + 1
}

example z for zeta {
  zeta(x: 1) == 2
}
```

## Limitations

- Wrapping a long comparison expression can make it harder to read. The goal of canonical form is determinism, not visual beauty.
- Clauses of the same kind remain in the order written, so two programs that differ only in the order of their `requires` clauses have the same meaning but different canonical text.
- The spelling of identifiers (snake_case, PascalCase) is enforced by the lexer, not by the formatter.

## Related

- Lexical rules: [02 Lexical structure](02-lexical-structure.md)
- Diagnostic codes `W0102` and `E0101`: [20 Diagnostic codes](20-diagnostics.md)
