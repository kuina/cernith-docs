# Interfaces

An **interface** is a bound that a program declares itself. It lists functions that a type promises to have, and it can state laws that every promise must keep. A type makes the promise with an explicit `impl`. A generic function asks for the promise with a bound, as in `<T: Priced>`.

Nothing is searched for and nothing is guessed. The `impl` that a call uses is chosen by the type of the argument, and an `impl` that does not exist is an error at the call (`E0316`).

## A first example

```cernith
module guide.priced;

import std.int;
import std.show;

pub record Book {
  title: String,
  pages: Int,
} where pages >= 0;

pub record Pen {
  color: String,
}

pub interface Priced {
  fn price(item: Self) -> Int;

  law never_negative(item: Self) {
    price(item: item) >= 0
  }
}

impl Priced for Book {
  fn price(item: Book) -> Int {
    item.pages * 2
  }
}

impl<T: Priced> Priced for List<T> {
  fn price(item: List<T>) -> Int {
    var total: Int = 0;
    for element in item {
      total = total + price(item: element);
    }
    total
  }
}

impl Priced for Pen {
  fn price(item: Pen) -> Int {
    150
  }
}

impl show.Show for Book derive;

impl show.Show for Pen derive;

pub fn bill<T: Priced>(items: List<T>) -> Int {
  price(item: items)
}

example bill_adds_the_prices for bill {
  bill(items: [Pen(color: "red"), Pen(color: "blue")]) == 300
}

pub fn label<T: Priced + show.Show>(item: T) -> String {
  show.text(value: item) ++ " costs " ++ int.to_text(value: price(item: item))
}

example label_names_the_item_and_its_price for label {
  label(item: Pen(color: "red")) == "Pen(color: \"red\") costs 150"
    and label(item: Book(title: "Cernith", pages: 10))
      == "Book(title: \"Cernith\", pages: 10) costs 20"
}
```

`Priced` is the interface. `Book` and `Pen` each promise it with an `impl`. `bill` and `label` are generic functions that work for every type that has made the promise. `cernith test` runs the example, and it also runs the law `never_negative` on generated `Book` and `Pen` values.

The rest of this chapter takes the parts one at a time.

## Declaring an interface

```text
[pub] interface Name {
  fn method(label: Type, ...) -> Type [uses effects];

  law name(label: Type, ...) {
    [requires condition;]
    body
  }
}
```

- `Self` stands for the type that implements the interface. It can be written only inside the interface.
- A **method signature** has a name, labeled parameters, a result type, and optionally `uses`. It has no body, and no `requires` or `ensures`. Laws say what is expected of a method.
- A method must mention `Self` in a parameter or in its result. A method without `Self` could not tell which type's `impl` to use (`E1401`).
- A method has no type parameters of its own, and an interface has no type parameters (`E1401`).
- A **law** is a property over values of `Self`. Its body is a `Bool` expression, and it calls the methods by their bare names. A `requires` line discards the cases it rejects, as in a `property` ([16 Testing](16-testing.md)).
- An interface is a name for a bound, not a type. `fn cost(item: Priced) -> Int` is an error (`E1405`). Write a type parameter: `fn cost<T: Priced>(item: T) -> Int`.

```cernith-error E1401
module guide.maker;

pub interface Maker {
  fn make() -> Int;
}
```

```cernith-error E1405
module guide.priced;

pub interface Priced {
  fn price(item: Self) -> Int;
}

pub fn cost(item: Priced) -> Int {
  1
}
```

`interface`, `impl`, `law`, and `derive` are not reserved words. A program that uses them as names keeps working. They are read as keywords only at the places where the grammar expects them.

## Implementing an interface

```text
impl path.Interface for Type {
  fn method(label: Type, ...) -> Type { ... }
}

impl<T: Bound> path.Interface for Type<T> { ... }
impl path.Interface for Type derive;
```

- `Type` is a named type: a record, a variant, a `distinct` or refined type, or a prelude type (`Int`, `String`, `List<T>`, `Option<T>`, ...). It is not a type parameter and not a function type.
- The methods are written without `pub`. An `impl` is never `pub` either: it is visible wherever its interface and its type are visible.
- Every type parameter of the `impl` occurs in `Type`. The bounds of the parameters are available in the method bodies. `impl<T: Priced> Priced for List<T>` is an `impl` for every list whose elements are `Priced`.
- The interface is written as `Name` in its own module and as `alias.Name` everywhere else, the same as any other name from a module.
- `derive;` asks the compiler to write the methods. It is available for `Show`, `Encode`, and `Decode`, which are described in [std.show](../stdlib/show.md) and [std.codec](../stdlib/codec.md).

An `impl` is checked against its interface (`E1403`). It has exactly the methods of the interface. Each method has the same labels and, with `Self` replaced by the type of the `impl`, the same parameter and result types. It may use at most the effects that the interface declares for the method. The body of a method is an ordinary function body, checked like any other.

```cernith-error E1403
module guide.priced;

pub record Pen {
  color: String,
}

pub interface Priced {
  fn price(item: Self) -> Int;
}

impl Priced for Pen {
  fn price(pen: Pen) -> Int {
    150
  }
}
```

## Calling a method

A method is a generic function of the module that declares the interface. In `guide.priced` it is `price`, and from another module that imports the module as `priced` it is `priced.price(item: x)`. It is called like every function, with labels. It has no receiver and is not a field of a value.

The type of the argument chooses the `impl`. `price(item: Pen(color: "red"))` uses the `impl` for `Pen`, and `price(item: [Pen(color: "red")])` uses the one for `List<T>`. A method can also be passed as a value, as in `apply: price`.

When `Self` occurs only in the result, no argument tells which `impl` to use. Write the type with `::<T>`, as in `fresh::<Int>(seed: 1)`. The standard library has such a method, `codec.decode::<Point>(value: v)`. If the context already fixes the type (for example, the call is the result of a function whose result type is `Int`), the compiler accepts the call without `::<T>`. If nothing fixes it, the error is `E0320` ([06 Functions](06-functions.md)).

```cernith
module guide.fresh;

pub interface Fresh {
  fn fresh(seed: Int) -> Self;
}

impl Fresh for Int {
  fn fresh(seed: Int) -> Int {
    seed * 2
  }
}

impl Fresh for String {
  fn fresh(seed: Int) -> String {
    "item"
  }
}

pub fn number() -> Int {
  fresh::<Int>(seed: 4)
}

example number_uses_the_int_impl for number {
  number() == 8
}

pub fn text() -> String {
  fresh::<String>(seed: 4)
}

example text_uses_the_string_impl for text {
  text() == "item"
}
```

```cernith-error E0320
module guide.fresh;

pub interface Fresh {
  fn fresh(seed: Int) -> Self;
}

impl Fresh for Int {
  fn fresh(seed: Int) -> Int {
    seed * 2
  }
}

pub fn drop_it() -> Int {
  discard fresh(seed: 4);
  0
}
```

## Bounds

A bound is `Eq`, `Ord`, or the name of an interface. A type parameter can have several bounds, joined by `+`: `<K: Ord + show.Show>`. A bound that is none of these is `E0318`. A private interface of another module counts as unknown.

```cernith-error E0318
module guide.keep;

pub fn keep<T: Priced>(item: T) -> T {
  item
}
```

A type satisfies a bound in one of three ways.

1. The type is a type parameter that has the bound in its own list (`Ord` also gives `Eq`).
2. An `impl` of the interface exists for the type's constructor, and the bounds of that `impl`'s own parameters hold for the type's arguments. `List<Pen>` is `Priced` because `Pen` is. `List<String>` is not, if `String` is not.
3. The type is a refined type that has no `impl` of its own, and its base type satisfies the bound. A `distinct` type is abstract, so it needs an `impl` of its own.

An unsatisfied bound is `E0316`. It is checked after type inference, and the message names the concrete type and says that no `impl` exists.

```cernith-error E0316
module guide.priced;

pub record Pen {
  color: String,
}

pub interface Priced {
  fn price(item: Self) -> Int;
}

impl Priced for Pen {
  fn price(item: Pen) -> Int {
    150
  }
}

pub fn cost<T: Priced>(item: T) -> Int {
  price(item: item)
}

pub fn cost_of_text() -> Int {
  cost(item: "pen")
}
```

Inside a function with the bound `T: Priced`, the methods of `Priced` can be called on values of type `T`. Nothing else is known about `T`. Calling a method on a `T` without the bound is `E0316`, as is `==` on a `T` that is not `Eq`.

```cernith-error E0316
module guide.priced;

pub interface Priced {
  fn price(item: Self) -> Int;
}

pub fn twice<T>(item: T) -> Int {
  price(item: item) * 2
}
```

## One impl for each type

For every interface and every type constructor there is at most one `impl` in the whole program. It is written in the module that declares the interface or in the module that declares the type. A prelude type such as `Int` or `List<T>` has no module, so only the module of the interface can implement for it. Because of this rule, two libraries cannot disagree about an instance, and no search through the imports is needed. Breaking the rule is `E1402`.

```text
shapes.cnth   module shapes;   pub interface Area { ... }   pub record Square { ... }
report.cnth   module report;   pub record Tile { ... }
```

`impl shapes.Area for report.Tile` may be written in `shapes` or in `report`. In a third module that imports both, it is `E1402`.

`List<Int>` and `List<String>` are the same type constructor, so they cannot have different `impl`s of one interface. A program that wants two behaviors writes two functions, or a wrapper type.

```cernith-error E1402
module guide.priced;

pub interface Priced {
  fn price(item: Self) -> Int;
}

impl Priced for List<Int> {
  fn price(item: List<Int>) -> Int {
    1
  }
}

impl Priced for List<String> {
  fn price(item: List<String>) -> Int {
    2
  }
}
```

An `impl` that is for a type parameter (`impl<T> Priced for T`), or whose type parameter does not occur in its type (`impl<T> Priced for Int`), is also `E1402`.

## Derived impls

`impl Interface for Type derive;` asks the compiler to write the methods. It is available for the interfaces that have a rule in the standard library (`std.show`, `std.codec`), and for a record or variant whose fields all satisfy the interface. Otherwise it is `E1404`, and the message names the first field that fails. For an interface without a rule, such as `Priced`, `derive` is also `E1404`.

```cernith-error E1404
module guide.pen;

import std.show;

pub record Color {
  name: String,
}

pub record Pen {
  color: Color,
}

impl show.Show for Pen derive;
```

Giving `Color` its own `impl show.Show for Color derive;` first fixes it. A generic type needs the bound on its parameters: `impl<T: show.Show> show.Show for Box<T> derive;`.

## Laws

A law is run for each `impl` of its interface that has no type parameters. `cernith test` instantiates the law at the type of the `impl` and runs it on generated values, like a `property` ([16 Testing](16-testing.md)). The test is named `law_` + the interface + the law + `for` + the type, as in `law_Priced_never_negative_for_Pen`. This includes the laws of interfaces from other modules: the law `stable` of `Show` is run for `Show` impls in your module.

A law that is false for an `impl` is a defect of the `impl`. The test fails and shows the counterexample.

```cernith-snippet
module guide.refunds;

pub interface Priced {
  fn price(item: Self) -> Int;

  law never_negative(item: Self) {
    price(item: item) >= 0
  }
}

pub record Refund {
  amount: Int,
}

impl Priced for Refund {
  fn price(item: Refund) -> Int {
    0 - item.amount
  }
}
```

```text
$ cernith test refunds.cnth
failed   property guide.refunds.law_Priced_never_negative_for_Refund  counterexample: item: Refund(amount: 1)
1 tests: 1 failed
```

`cernith test --only never_negative` runs only the tests whose name contains the text. The verifier does not prove laws in this version.

## Eq and Ord

`Eq` and `Ord` stay as they are: `==` and `<` need no `impl`, and a user `impl Eq` or `impl Ord` is `E1402`.

```cernith-error E1402
module guide.order;

pub record Version {
  major: Int,
}

impl Ord for Version derive;
```

A record therefore still cannot be a `Map` key or a `Set` element. An interface can say what an order means for a type (an interface with a `compare` method), and a program can use it in its own functions, but `Ord` itself is not changed by it.

## Not in this version

- Default methods, and an interface that extends another
- Associated types
- Type parameters on interfaces and on methods, and arguments on a bound
- User `impl`s of `Eq` and `Ord`
- `impl`s for function types
- Proofs of laws by the verifier (laws are tested)

## Diagnostic codes

| Code | Name | Meaning |
| --- | --- | --- |
| `E0316` | bound-not-satisfied | A type that must satisfy `Eq`, `Ord`, or an interface has no `impl` of it |
| `E0318` | unknown-bound | A bound that is neither `Eq`, `Ord`, nor an interface in scope |
| `E1401` | invalid-interface | A method without `Self`, a method with type parameters, or type parameters on the interface |
| `E1402` | incoherent-impl | A second `impl` for the interface and type constructor, an `impl` outside the two modules, an `impl` for something that is not a named type, a type parameter that does not occur in the type, or an `impl` of `Eq` or `Ord` |
| `E1403` | impl-does-not-match | A missing or extra method, other labels or types, or more effects than the interface declares |
| `E1404` | cannot-derive | `derive` for an interface without a rule, for a type that is not a record or variant, or for a field that does not satisfy the interface |
| `E1405` | interface-is-not-a-type | An interface written where a type is expected |

An `impl` written with `pub` is a syntax error (`E0001`).

## Related

- Generic functions and the bounds `Eq` and `Ord`: [06 Functions](06-functions.md)
- Laws as tests: [16 Testing](16-testing.md)
- `Show`: [std.show](../stdlib/show.md); `Encode` and `Decode`: [std.codec](../stdlib/codec.md)
- Visibility: [14 Modules and packages](14-modules-and-packages.md)
