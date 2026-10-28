# Sokoban

A text-based Sokoban puzzle written as twelve small Cernith modules. All game logic
is implemented in pure functions; a small `main` reads standard input and prints
boards. This larger sample in `examples/` shows how to organize a program with
contracts, laws, and end-to-end examples across multiple modules.

You push boxes (`$`) onto goals (`.`). You can only push, one box at a time, and never
into a wall or another box. When every box is on a goal the level is solved.

## How to run

Run from the documentation repository root, supplying moves through
standard input, one command per line. Each command is a sequence of letters.
The program prints a board after each line.

```text
cernith run examples/larger/sokoban < examples/larger/sokoban/run.stdin
cernith run examples/larger/sokoban --args 2 < my_moves.txt
```

`--args 2` picks level 2 (the default is level 1; there are five). A bad number gives
a message and exit status 2. To run the same checks as `tools/check_examples.py`:

```text
cernith fmt --check examples/larger/sokoban
cernith check --release examples/larger/sokoban
cernith check --verify examples/larger/sokoban
cernith test examples/larger/sokoban
```

## Controls

Each input line is a string of letters, applied in order.

| Letter | Meaning |
|---|---|
| `u` `d` `l` `r` | move up, down, left, right (pushes a box if one is in the way and the cell behind it is free) |
| `z` | undo the last move or push (unbounded history) |
| `x` | restart the level (clears the undo history) |

Anything else prints `unknown move 'q' skipped` and is skipped. A move into a wall, or a
push that cannot happen, changes nothing and is not counted. Once the level is solved
the rest of the input is ignored, with a note.

Board symbols (the same in the level texts and in the output): `#` wall, space floor,
`.` goal, `$` box, `*` box on a goal, `@` player, `+` player on a goal.

## A real run

This is the output of `run.stdin` (the five lines `uq`, `l`, `z`, `ldlu`, `ud`) on level 1,
copied from the program:

```text
Level 1: Warm-up
######
#. . #
# $$ #
#  @ #
######
moves: 0, pushes: 0

> uq
unknown move 'q' skipped
######
#. * #
# $@ #
#    #
######
moves: 1, pushes: 1

> l
######
#. * #
#$@  #
#    #
######
moves: 2, pushes: 2

> z
######
#. * #
# $@ #
#    #
######
moves: 1, pushes: 1

> ldlu
######
#* * #
#@   #
#    #
######
moves: 5, pushes: 3 - SOLVED

> ud
solved: ignoring the last 2 move(s) of this line
######
#* * #
#@   #
#    #
######
moves: 5, pushes: 3 - SOLVED
```

Level 2 (`--args 2`) with its stored solution `lluruulldrlldrldr` ends like this:

```text
 ######
##    ##
#   *  #
#  * * #
# @*   #
##    ##
 ######
moves: 17, pushes: 6 - SOLVED
```

## Module map

Modules are listed roughly in dependency order. `samples` provides test fixtures for `moves` and the modules above it. Only `main` declares `uses io`.

| Module | Role |
|---|---|
| `point` | `Point` (row, col), `Direction`, the letters `u d l r`, `shift` with a proved postcondition |
| `board` | the fixed part of a puzzle: `Tile` (wall, floor, goal), `Board`, safe lookups (outside is a wall) |
| `level` | parses level text into a validated `Level`: `Result` with a structured `LevelError`; the `Level` record has the invariant "as many boxes as goals" |
| `moves` | the rules: `State`, `step` returns `Blocked`, `Moved` or `Pushed` with a contract on the counters, `move_box`, `walk` |
| `history` | a generic `History<T>` stack: `remember` and `undo` |
| `render` | board and status line as lists of strings |
| `game` | `Game` (state plus undo trail), commands `Move`, `Undo`, `Restart`, and `parse_command` |
| `session` | `apply_line` (a `while` loop with `decreases`, stops when solved) and `play` (a whole input as output lines) |
| `levels` | the built-in levels and their solutions as data; examples play every solution |
| `samples` | three small levels shared by the examples and properties of the other modules |
| `laws` | properties about `step` run on random walks: one push moves one box one cell, a blocked step changes nothing, a move can be walked back, the box count never changes |
| `main` | choosing the level, `execute` (arguments and input in, lines and status out, pure), and the `io` shell `main` |

## What to read first

1. `main.cnth` (the shape of the whole program, and the end-to-end examples that run a game
   as lists of strings, with and without a simulated terminal).
2. `moves.cnth`, `step`: the whole rule set and its contract, which `check --verify` proves.
3. `level.cnth`, `parse` and `build`: errors as values, `?`, and a record invariant that the
   checker proves at the one place where the record is built.
4. `session.cnth`, `apply_line`: a `while` loop with `invariant` and `decreases`.
5. `laws.cnth`: properties that state what no contract says directly.
6. `history.cnth`: a small generic module with a law (`undo` after `remember`).
