# Proof obligations and assurance levels

From contracts ([09 Contracts](09-contracts.md)) and from the places where a program can stop with a defect (an out-of-range index, division by zero, and the like), the checker creates **proof obligations** and attempts to prove them. Proofs are performed with `cernith check --verify`. `cernith check` without `--verify` checks the rules for syntax, names, types, effects, exhaustiveness, and termination, but does not examine proof obligations.

## Proof obligations

A proof obligation is a claim that the checker must establish: “at this place, this condition holds.” Each one has a kind, a location, and a party that bears the responsibility.

| Kind | Where it arises | What it shows | Responsibility | Diagnostic code (when refuted) |
| --- | --- | --- | --- | --- |
| `pre` | Each call | The `requires` of the called function | The caller | `E0501` |
| `post` | The end of a function and each `return` | Each `ensures` | The function | `E0502` |
| `restrict` | Construction of a refined type or of a record with an invariant, and `with` | The condition holds | The place that constructed the value | `E0503` |
| `assert` | Each `assert` | The asserted expression | The function | `E0504` |
| `loop-invariant` | Each loop | The invariant holds on entry and after one iteration | The function | `E0505` |
| `index` | Each `xs[i]` | `0 <= i < xs.length` | The function | `E0506` |
| `float-div` | Each `/` on `Float64` | The divisor is not 0 | The function | `E0507` |
| `float-range` | Each `+ - * /` on `Float64` | The result is finite | The function | `E0508` |
| `term` | Recursive calls and loops with `decreases` | The decreasing measure is non-negative and becomes smaller | The function | `E0509` |

Standard library functions that can cause a defect, such as integer division and `float.sqrt`, are treated as having a `requires` of their own (an obligation of kind `pre`). Type correctness, the rules for effects, and the exhaustiveness of `match` are static checks, not proof obligations.

### How obligations are created

Obligations are created by symbolic execution of function bodies. This is done **modularly**: the body of a function is examined using only **its own contract** and **the contracts of the functions it calls**. The bodies of the called functions are not looked at.

1. **Entry.** Each parameter is an arbitrary value of its type. The facts that hold are the conditions of parameters of refined types and the function’s `requires`.
2. **Straight-line code.** An obligation is examined under all the facts and branch conditions that hold at that point. After an obligation has been proved (or when execution passes a place that could cause a defect), the analysis continues with that claim assumed to hold.
3. **Branches.** In `if c { a } else { b }`, the obligations in `a` are examined assuming `c`, and the obligations in `b` assuming `not c`. `match` works the same way with the conditions of the patterns.
4. **Short-circuiting.** The obligations in the `b` of `a and b` are examined assuming `a`, those in the `b` of `a or b` assuming `not a`, and those in the `b` of `a implies b` assuming `a`.
5. **Calls.** The `requires` of the called function becomes an obligation of kind `pre`. The return value is an unknown value, but it is assumed to satisfy the condition of the return type and the `ensures` of the called function. For the return value of a function without `ensures`, nothing is assumed.
6. **Recursion.** A call to the function itself is an ordinary call. If there is a `decreases M`, an obligation of kind `term`, `0 <= M(arguments) < M(parameters)`, is also created.
7. **Loops.** For a loop with `invariant I` and `decreases M`, the checker shows that `I` holds on entry, and that after one iteration starting from `I` and the loop condition, `I` holds again and `M` has decreased. After the loop, `I` and the negation of the loop condition are assumed to hold, and nothing else is known about the `var`s assigned in the loop. A loop without an invariant also turns the assigned `var`s into unknown values, so obligations that depend on them can no longer be decided.
8. **Early exit.** At `return e;`, the obligation of kind `post` is examined at that point.

`Float64` values are assumed to be finite.

## States

An obligation ends up in one of the following states.

| State | Meaning |
| --- | --- |
| Proved (`proved`) | It was shown to hold. No runtime check is needed |
| Refuted (`refuted`) | An input for which it does not hold actually exists. It is reported together with the counterexample input |
| Unknown (`unknown`) | The solver could not decide (for example, a timeout). A runtime check remains |
| Tested only (`tested-only`) | An executable condition that held in all tests, but has no proof |
| Not attempted (`not-attempted`) | A proof-only condition that has not been proved, or the condition contains syntax the checker cannot handle |

**`refuted` is reported only when the counterexample actually exists.** That is the case when the representation of every path is exact and contains no approximation, or when the function could actually be run on the counterexample input and the failure confirmed. On a path where the contract of a called function was used as an approximation, the counterexample may not actually exist, so the state is `unknown` and the counterexample is marked as a “candidate.” `unknown` shows the budget (`rlimit`) that was used.

| State | Severity |
| --- | --- |
| Refuted | Error (`E05xx`). The counterexample input and the responsibility are attached. However, `float-range` (`E0508`) is treated as an error only for definitions whose minimum level is `safe` (by default, `pub` definitions); for other definitions it is reported, but the check does not fail |
| Unknown | Warning `W0504` |
| Tested only, not attempted | Note (neither an error nor a warning) |

The counterexample input is written in the form of Cernith code. The following is a function whose postcondition is broken when `value` is negative.

```cernith-error E0502
module guide.refuted;

pub fn absolute(value: Int) -> Int {
  ensures result >= 0;

  value
}
```

```text
t.cnth:6:3 E0502 postcondition-violated: The body can return a value that violates its postcondition. [refuted, blame callee; value=-1]
```

The following are examples in which the obligations for `xs[i]`, `assert`, a loop invariant, and `Float64` division are refuted.

```cernith-error E0506
module guide.refuted;

pub fn at(xs: List<Int>, i: Int) -> Int {
  xs[i]
}
```

```cernith-error E0504
module guide.refuted;

pub fn positive(n: Int) -> Int {
  requires n >= 0;

  assert n > 0;
  n
}
```

```cernith-error E0505
module guide.refuted;

pub fn count_up(n: Int) -> Int {
  requires n >= 0;

  var i: Int = 0;
  while i < n {
    invariant i < 0;
    decreases n - i;

    i = i + 1;
  }
  i
}
```

```cernith-error E0507
module guide.refuted;

pub fn ratio(a: Float64, b: Float64) -> Float64 {
  requires a >= -1.0e100 and a <= 1.0e100;

  a / b
}
```

The following is an example that the solver cannot decide. To show that `result <= a` also holds in the `else` branch, the solver has to reason about the order of strings, and it gives up within its budget, so the following postcondition is `unknown` (`W0504`). A runtime check remains, so if the contract is broken, the program stops with a defect at run time.

```cernith-error W0504
module guide.unknown;

pub fn smaller(a: String, b: String) -> String {
  ensures result <= a and result <= b;

  if a < b {
    a
  } else {
    b
  }
}
```

> Note: There are syntax forms and types that the checker cannot use in proofs. They are function expressions (closures), recursive `spec fn`s, `Map`, and types that contain a list of themselves (for example, a `Node` that has a case `Branch(children: List<Node>)`). Obligations that involve these are neither proved nor refuted, and become runtime checks.

## Assurance levels

For each definition, the checker reports the **assurance level**, as the highest one that has been reached.

| Level | Requirement |
| --- | --- |
| `typed` | Passes type checking, the rules for effects are kept, and `match` is exhaustive |
| `safe` | All obligations of kinds `pre`, `restrict`, `assert`, `index`, `float-div`, and `float-range` are proved |
| `specified` | In addition to `safe`, there is at least one `example`, and at least one `ensures` or property that is not vacuous (described below) |
| `proved` | In addition to `specified`, all obligations for each `ensures`, `loop-invariant`, and `term` are proved |

The levels appear, per definition, in `summary.levels` of the JSON output of `cernith check --verify`.

```cernith
module guide.levels;

pub fn add(first: Int, second: Int) -> Int {
  first + second
}

example add_adds for add {
  add(first: 1, second: 2) == 3
}

pub fn clamp(value: Int, low: Int, high: Int) -> Int {
  requires low <= high;
  ensures result >= low and result <= high;

  if value < low {
    low
  } else if value > high {
    high
  } else {
    value
  }
}

example clamp_above for clamp {
  clamp(value: 12, low: 0, high: 9) == 9
}

pub fn first_of(items: List<Int>) -> Int {
  requires items.length > 0;

  items[0]
}

example first_of_reads_the_head for first_of {
  first_of(items: [7, 8]) == 7
}
```

When `cernith check --verify` is run on the module above, `summary.levels` contains the following. `add` has no contract and no obligation left to prove, so it is `safe`; `first_of` is also `safe`, because the index is protected by the precondition; and `clamp` is `proved`, because all of its obligations are proved.

```text
"levels": {
  "guide.levels::add": "safe",
  "guide.levels::clamp": "proved",
  "guide.levels::first_of": "safe"
}
```

## Vacuous contracts

A contract that always holds guarantees nothing. The checker warns about the following postconditions with `W0801` (vacuous-contract) and does not allow the definition to be raised to `specified` by such a contract.

- A postcondition that follows **from the types alone** (one that holds for any body).
- A postcondition that **merely repeats the body** (`result == value + value` when the body is also `value + value`).
- A postcondition that continues to hold even when the body is mutated (operators swapped, constants changed).

```cernith-error W0801
module guide.vacuous;

pub fn twice(value: Int) -> Int {
  ensures result == value + value;

  value + value
}
```

```cernith-error W0801
module guide.vacuous;

pub fn identity(value: Int) -> Int {
  ensures result >= value or result < value;

  value
}
```

With `--no-vacuity`, this detection is skipped.

## Relation to run time

Conditions that have been proved are not checked at run time. Conditions that cannot be proved and are executable are checked at run time ([09 Contracts](09-contracts.md)). In other words, an undecided proof does not mean that the safety of the program is lost; it means that the check moves to run time.

## Related

- How to write contracts: [09 Contracts](09-contracts.md)
- Defects: [12 Errors and defects](12-errors-and-defects.md)
- Termination (obligations of kind `term`): [13 Termination](13-termination.md)
