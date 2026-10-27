# Units of measure

A number can carry a unit, and the checker keeps units apart. Adding a length to a time is an error, and dividing a distance by a time gives a speed. Units exist only in the checker: the verifier and every target see the plain `Float64` or `Int`, so a unit costs nothing at run time.

## Declaring a unit

```text
[pub] unit name;
```

`unit m;` declares a base unit in the module. Two units are different types even when they measure the same thing: `m` and `ft` are never mixed without a conversion. A `pub` function or type that mentions a unit needs the unit to be `pub` as well (`E0216` otherwise). A unit of another module is written with the alias of the import, as in `units.m` after `import si.units;`.

```cernith-error E0216
module guide.errors;

unit m;

pub fn read(length: Float64<m>) -> Float64 {
  length.value
}
```

## Quantity types

`Float64<UNIT>` and `Int<UNIT>` are types of numbers with a unit. A type without a unit is the plain `Float64` or `Int`. `UNIT` is written in one of these forms.

| Form | Example | Meaning |
| --- | --- | --- |
| One unit | `m` | A base unit |
| Product | `m * s` | A product of units |
| Quotient | `m / s` | A unit over a unit |
| Power | `m / s^2` | A power is written with `^` and a whole number of 2 or more |
| Several factors below the bar | `kg / (m * s)` | The denominator is in parentheses |
| Inverse | `1 / s` | A unit with only a denominator |

### Canonical form

A unit has exactly one text. A unit that is written in another way is `E0362`, and the message gives the canonical text.

- The factors on each side are sorted by name (`m * s`, not `s * m`).
- A power is written once (`m^2`, not `m * m`).
- There is one bar, and a denominator of several factors is in parentheses.
- A unit that cancels out (`m / m`) is the plain type.

```cernith-error E0362
module guide.errors;

pub unit m;

pub fn area(width: Float64<m * m>) -> Float64<m^2> {
  width
}
```

## Making and reading a quantity

The constructor has the name of the unit. `m(value: 5.0)` has the type `Float64<m>`, and `m(value: 5)` has the type `Int<m>`. `length.value` is the plain number of a quantity. Both are the identity at run time.

| Expression | Meaning |
| --- | --- |
| `m(value: 5.0)` | A `Float64<m>` from the plain number `5.0` |
| `m(value: 5)` | An `Int<m>` from the plain number `5` |
| `length.value` | The plain number of a quantity |

The argument of a constructor must be an `Int` or a `Float64` (`E0361`), and a quantity is not made again from a quantity (`E0360`): read its number with `.value` first.

```cernith-error E0361
module guide.errors;

pub unit m;

pub fn name() -> Float64<m> {
  m(value: "far")
}
```

```cernith-error E0360
module guide.errors;

pub unit m;

pub fn again(length: Float64<m>) -> Float64<m> {
  m(value: length)
}
```

## Arithmetic

| Expression | Rule |
| --- | --- |
| `a + b`, `a - b` | The units of both sides must be equal. The result has that unit |
| `a == b`, `a < b`, and the other comparisons | The units of both sides must be equal. The result is a `Bool` |
| `-a` | Keeps the unit |
| `a * b` | The units are multiplied (`m * m` is `m^2`) |
| `a / b` | The units are divided (`m / s`). If both are the same unit, the result is a plain number |
| A plain number times or over a quantity | Scales the quantity and keeps its unit (`2.0 * length`, `length / 2.0`) |
| A plain number over a quantity | Has the inverse unit (`1.0 / time` has the unit `1 / s`) |

`Int` and `Float64` do not mix, with or without a unit (`E0302`). `Int / Int` is `E0307` as for plain integers, so a quantity with a denominator is made from `Float64`. The `Int` example below uses a product and the same unit only.

```cernith
module guide.areas;

pub unit frames;

pub unit m;

pub fn area(width: Int<m>, height: Int<m>) -> Int<m^2> {
  width * height
}

example area_multiplies_the_units for area {
  area(width: m(value: 3), height: m(value: 4)).value == 12
}

pub fn doubled(length: Int<m>) -> Int<m> {
  2 * length
}

example doubled_scales_the_length for doubled {
  doubled(length: m(value: 5)).value == 10
}

pub fn later(first: Int<frames>, second: Int<frames>) -> Bool {
  first > second
}

example later_compares_the_same_unit for later {
  later(first: frames(value: 30), second: frames(value: 20))
}

pub fn total(first: Int<m>, second: Int<m>) -> Int<m> {
  first + second
}

example total_adds_the_same_unit for total {
  total(first: m(value: 2), second: m(value: 3)) == m(value: 5)
}
```

```cernith-error E0302
module guide.errors;

pub unit m;

pub fn mixed(whole: Int<m>, part: Float64<m>) -> Float64<m> {
  part + whole
}
```

A number with a different unit, or with none, where a unit is expected is `E0360` (unit-mismatch). The message writes both types. There is no implicit conversion, not even between metres and feet. Two fixes are offered, and both need review: write `m(value: ...)` around a plain number, or add `.value` to a quantity where a plain number is expected.

```cernith-error E0360
module guide.errors;

pub unit ft;

pub unit m;

pub fn total(first: Float64<m>, second: Float64<ft>) -> Float64<m> {
  first + second
}
```

```cernith-error E0360
module guide.errors;

pub unit m;

pub fn longer(length: Float64<m>) -> Float64<m> {
  length + 1.0
}
```

## Conversions and library functions

A conversion between units is a function that the program writes. It reads the number with `.value` and makes the new quantity with the constructor. The factor is a plain number.

Library functions take plain numbers. Write `.value` for the argument and make the result a quantity again.

```cernith-error E0360
module guide.errors;

import std.float;

pub unit m;

pub fn root(area: Float64<m^2>) -> Float64<m> {
  m(value: float.sqrt(value: area))
}
```

A quantity does not cross the host boundary: a `foreign` function cannot take or return one (`E0213`). Wrap the call in a Cernith function that converts with `.value` and the constructor.

```cernith-error E0213
module guide.errors;

pub unit m;

foreign python "math.sqrt";
pub fn root(x: Float64<m>) -> Float64<m> {
  requires x.value >= 0.0;
}
```

## Verification

The solver sees the plain numbers, so a contract speaks about `.value` (`ensures result.value == ...`). It does not decide `Float64` arithmetic in general: a `Float64` operation whose result might be infinite is reported by `cernith check --verify` as `E0508`, and one it cannot decide as `W0504` ([03 Types](03-types.md), [10 Proof obligations and assurance levels](10-verification.md)). Narrow the inputs with `requires`, as in the following example.

## Example

```cernith
module guide.motion;

import std.float;

pub unit ft;

pub unit m;

pub unit s;

pub fn is_faster(first: Float64<m / s>, second: Float64<m / s>) -> Bool {
  first > second
}

example is_faster_compares_speeds for is_faster {
  is_faster(
    first: m(value: 5.0) / s(value: 1.0),
    second: m(value: 3.0) / s(value: 1.0),
  )
}

pub fn side_of(area: Float64<m^2>) -> Float64<m> {
  requires area.value >= 0.0 and area.value <= 1000000.0;

  m(value: float.sqrt(value: area.value))
}

example side_of_takes_the_root for side_of {
  side_of(area: m(value: 3.0) * m(value: 3.0)).value == 3.0
}

pub fn speed(distance: Float64<m>, time: Float64<s>) -> Float64<m / s> {
  requires distance.value >= 0.0 and distance.value <= 1000000.0;
  requires time.value >= 1.0 and time.value <= 1000000.0;

  distance / time
}

example speed_divides_the_units for speed {
  speed(distance: m(value: 100.0), time: s(value: 4.0)).value == 25.0
}

pub fn to_metres(length: Float64<ft>) -> Float64<m> {
  requires length.value >= 0.0 and length.value <= 1000000.0;

  m(value: length.value * 0.3048)
}

example to_metres_converts for to_metres {
  to_metres(length: ft(value: 100.0)).value == 30.48
}

pub fn travelled(speed: Float64<m / s>, time: Float64<s>) -> Float64<m> {
  requires speed.value >= 0.0 and speed.value <= 1000.0;
  requires time.value >= 0.0 and time.value <= 1000000.0;

  speed * time
}

example travelled_multiplies_the_units for travelled {
  travelled(speed: m(value: 25.0) / s(value: 1.0), time: s(value: 4.0)).value
    == 100.0
}
```

## Related

- Numbers: [03 Types](03-types.md)
- Types that keep values apart: [08 Refined types and distinct types](08-refined-and-distinct-types.md)
- Public declarations of a module: [14 Modules and packages](14-modules-and-packages.md)
- Foreign functions: [15 Foreign functions](15-foreign-functions.md)
- Functions of `Float64`: [std.float](../stdlib/float.md)
