# expense_report

A command-line tool that reads a CSV file of expenses from standard input and prints a report: totals per category and per month, the largest expenses, budget warnings and a grand total. It is a larger sample (eleven small modules, one of them test support) that shows how a real Cernith program is put together: all logic in pure functions with contracts, examples and properties, and one thin `uses io` shell.

Money is parsed into integer cents and never touches `Float64`. Validation collects every problem instead of stopping at the first, and the report is still printed from the valid lines.

## How to run

From the root of the documentation repository (this is also what the example checker does, with `run.stdin` as standard input):

```
cernith run examples/larger/expense_report --args --month 2026-10 --budget food=300.00 < examples/larger/expense_report/expenses.csv
```

Check and test the package:

```
cernith fmt --check examples/larger/expense_report
cernith check --release examples/larger/expense_report
cernith check --verify examples/larger/expense_report
cernith test examples/larger/expense_report
```

## Options

Options come after `--args` and are read from left to right. Options that take a value are written with the value as the next argument.

| Option | Meaning |
| --- | --- |
| `--month YYYY-MM` | Report only the expenses of that month. A bad month is an option error. |
| `--budget CATEGORY=AMOUNT` | Print a warning when the category total is above `AMOUNT` (`food=300.00`). May be repeated, once per category. The category must be one of the known categories. |
| `--top N` | How many of the largest expenses to list (0 to 100, default 3). |
| `--strict` | Exit with status 1 when the input had problems (see below). |

An unknown option, a missing value, or a bad value prints `error: ...` and the usage text, and exits with status 2. Nothing else is printed in that case.

### Exit status

| Status | When |
| --- | --- |
| 0 | The report was printed. Problems in the data are listed after the report but do not change the status. |
| 1 | `--strict` was given and at least one line of the input was not a valid record (or the input could not be read or written). |
| 2 | The arguments were wrong. |

## The CSV format

```
date,category,description,amount
2026-10-03,food,Lunch with team,12.50
```

- The first line must be the header `date,category,description,amount`.
- `date` is `YYYY-MM-DD` and is checked for real: days per month and leap years (`2026-02-30` is an error, `2024-02-29` is fine).
- `category` is one of `food`, `lodging`, `office`, `other`, `software`, `travel` (upper case is accepted and lowered).
- `description` is not empty.
- `amount` has exactly two decimals or none: `12.50`, `7`. A leading `-` marks a refund (`-45.00`). `12.5`, `12.`, `.5`, `+4` and `1,5` are errors.
- Spaces around a field are ignored. Blank lines are skipped, but still counted in the line numbers.
- Quoted fields are NOT supported. A double quote anywhere in a line is an error, and so is a comma inside a field: it simply looks like one field too many.

## A real run

`expenses.csv` (the same text is in `run.stdin`) has 17 records, 4 of them bad. With the options of the command above:

```
Expense report for 2026-10
10 expenses in 5 categories

By category
  food      242.35  39%
  lodging   189.90  31%
  office     77.75  13%
  travel     75.00  12%
  software   29.99   5%

By month
  2026-10  614.99

Largest expenses
  1. 2026-10-01  Hotel in Berlin   189.90
  2. 2026-10-02  Train to Berlin   120.00
  3. 2026-10-21  Weekly groceries  102.35

Grand total: 614.99

Problems (4)
  line 9: amount "12.5x" is not a money amount (digits with two decimals or none)
  line 11: date "2026-02-30" has the day 30, but that month has 28 days
  line 12: unknown category "snacks"
  line 14: record has 5 fields, expected 4 (a comma inside a field is not supported)
```

The exit status was 0. With a tighter budget, `--budget food=200.00 --top 2 --strict`, the sections change like this and the status is 1:

```
Largest expenses
  1. 2026-10-01  Hotel in Berlin  189.90
  2. 2026-10-02  Train to Berlin  120.00

Budget warnings
  food: spent 242.35 of 200.00, over by 42.35

Grand total: 614.99
```

An unknown option is refused with status 2:

```
error: unknown option "--verbose"
usage: expense_report [options] < expenses.csv
  --month YYYY-MM            only expenses of that month
  --budget CATEGORY=AMOUNT   warn when a category exceeds its budget (repeatable)
  --top N                    list the N largest expenses (0 to 100, default 3)
  --strict                   exit with status 1 if the data had problems
```

(These outputs were produced by running this package with cernith 2.0.0, compiler commit `aae4ddcf`, on `expenses.csv`.)

## Module map

All modules are under `src/expense_report/`. Only `main` has `uses io`.

| Module | Role |
| --- | --- |
| `money` | `Cents`, an abstract `distinct` type: parsing `12.50` into cents without `Float64`, formatting, addition, rounded percentages. |
| `date` | `Date` and `Month` records with invariants, leap years, `days_in` (a `spec fn`) and `month_length` (its executable twin), parsing and formatting. |
| `csv` | Numbered lines, header skipping, splitting into exactly N fields, the two CSV errors. |
| `expense` | The typed `Expense`, the structured error variant `Reason` with the line number in `Problem`, header and category checks, `parse_row`. |
| `options` | Argument parser: `Options`, `Budget`, `OptionError`, `parse`, `usage`. |
| `aggregate` | `Map`- and `Set`-based totals, `sort_by` rankings, month filter, budget overspends. |
| `table` | Column widths found from the data: `widest`, `align_left`, `align_right`. |
| `report` | Renders a `Summary` as lines of text, and the problem list. |
| `app` | `run(lines, args) -> Outcome(lines, status)`, the whole program as a pure function, with end-to-end examples. |
| `fixtures` | Fake expenses shared by the examples and properties of several modules. |
| `main` | The thin shell: `io.read_all_lines`, `app.run`, `io.write_line`, the exit status. |

## What to read first

1. `main.cnth`: the shell, and how `test.with_world` tests it.
2. `app.cnth`: `run` and its end-to-end examples show every behavior (options, month filter, budgets, problems, `--strict`, refunds).
3. `money.cnth` and `date.cnth`: an abstract `distinct` type, record invariants with a `spec fn`, error variants instead of defects, and the properties (formatting then parsing returns the same cents; a valid date has a day between 1 and the month length).
4. `aggregate.cnth`: maps, sets, stable sorting, and the property that category totals add up to the grand total.
5. `options.cnth`: the argument parser, written as a recursion on the remaining arguments.

## Things to notice

- Errors are values. A bad line is a `Problem` (line number plus a `Reason` variant), not a defect, so the program can list all of them.
- Contracts that do work: `csv.fields` promises the field count, so `expense` indexes the fields without checking again; `align_left` and `align_right` promise the same number of cells, so `report` indexes them safely.
- `main` is the only function with `uses io`; everything else is tested with plain `example` and `property` items.
