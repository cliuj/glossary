# Part 1 — Reading Rust

[← Overview](00-overview.md) · [Next: Ownership and borrowing →](02-ownership-and-borrowing.md)

The goal here is narrow but valuable: after this part you can *read* any line of
basic Rust — parse what it's doing structurally — even when you don't yet know
what every function does. We won't explain ownership, lifetimes, or traits
properly; those get their own parts. This is the surface grammar, mapped piece by
piece onto the Go you already write.

---

## 1. Functions

A Rust function is `fn`, a name, parenthesized parameters, an arrow `->` to the
return type, and a brace body.

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

```go
// Go
func add(a int, b int) int {
    return a + b
}
```

Three differences. The keyword is `fn`, not `func`; the return type comes after
`->`; and — the load-bearing one — there is no `return` and no semicolon on
`a + b`, because the last expression in a block *is* the block's value (§3).
Parameter types go after the name, `a: i32`, the same order as Go's `a int`. A
function returning nothing omits the arrow; its return type is the unit type
`()`, the rough analog of no return values in Go.

### Multiple returns

Go hands back several values with `return a, b`. A Rust function returns exactly
one value, so "multiple returns" is a single **tuple**:

```rust
fn min_max(xs: &[i32]) -> (i32, i32) {
    let mut lo = xs[0];
    let mut hi = xs[0];
    for &x in xs {
        if x < lo { lo = x; }
        if x > hi { hi = x; }
    }
    (lo, hi)
}

let (a, b) = min_max(&[3, 1, 4, 1, 5]); // a => 1, b => 5
```

In Go you'd write `func minMax(...) (int, int)` and `return lo, hi`, then
`a, b := minMax(...)`. The Rust call site is identical (`let (a, b) = ...`
destructures like `a, b :=`), but mechanically it's one tuple value, not two
results. That matters later: a tuple is a real type you can store, pass, and put
in a `Vec`. Go's multiple return values are not a value you can name.

---

## 2. `let`, `let mut`, and shadowing

Bindings use `let`, and the biggest readability shift from Go is that **`let` is
immutable by default.**

```rust
let x = 5;
x = 6;          // compile error: cannot assign twice to immutable variable
let mut y = 5;
y = 6;          // fine — `mut` opts in to mutation
```

```go
// Go
x := 5
x = 6 // always fine; Go has no immutable locals
```

In Go every local is mutable. In Rust plain `let` is closest to a `const`-that-
can-hold-any-value, and `mut` is the escape hatch — so `mut` is a signal that a
binding is *expected* to change, and its absence is information. Types are
inferred like `:=`; annotate only when needed:

```rust
let count = 0;                  // inferred i32
let ratio: f64 = 0.5;           // annotated
let ids: Vec<u64> = Vec::new(); // annotation drives the element type
```

### Shadowing

You can re-`let` the same name. This is **not** mutation — it's a new binding
reusing the name, and it can even change the type:

```rust
let spaces = "   ";          // &str
let spaces = spaces.len();   // usize — same name, new binding, new type
println!("{spaces}");        // prints: 3
```

This has no Go equivalent — `:=` in the same scope is a redeclaration error, and
reassignment can't change type. Shadowing lets you transform a value through
stages without inventing `spaces_str`, `spaces_len`.

> **Trap:** `let mut x = 5; x = 6;` mutates one binding; `let x = 5; let x = 6;`
> declares a *second* binding. The second works without `mut` because you're not
> assigning — you're re-declaring. Don't read a repeated `let x` as an error just
> because the first wasn't `mut`.

---

## 3. Expressions, statements, and the semicolon

This is the section that makes the rest of Rust readable. Go is
**statement-oriented**: `if`, `for`, and switch *do* things, they don't have
values. Rust is **expression-oriented**: almost everything produces a value, and
a block `{ ... }` is itself an expression whose value is its final expression.

```rust
let y = {
    let a = 2;
    let b = 3;
    a + b      // no semicolon -> this is the block's value
};
println!("{y}"); // prints: 5
```

That trailing line has no `;`, so it's the block's value. Add a `;` and it
becomes a *statement* producing `()` (unit), so the block would evaluate to unit
and `y` would too — usually a type error. The semicolon is the difference between
"this is the value" and "this is just a step."

### `if` is an expression

```rust
let label = if count > 0 { "some" } else { "none" };
```

```go
// Go
var label string
if count > 0 {
    label = "some"
} else {
    label = "none"
}
```

Go has no ternary and `if` is a statement, so you declare first and assign per
branch. In Rust `if/else` *is* a value — which is why Rust needs no ternary
operator. Both arms must produce the same type.

### `match` is an expression too (full treatment in Part 4)

Read `match` as a value-producing, *exhaustive* switch:

```rust
let kind = match count {
    0 => "zero",
    1 => "one",
    _ => "many",     // `_` is the catch-all, like Go's `default`
};
```

At the reading level: each arm `pattern => value` yields a value, and `match`
must cover every case — the compiler rejects a `match` with a hole. The deep
treatment (binding, enums, `Option`/`Result`) is Part 4.

### `loop`/`while`/`for` (full treatment in Part 7)

`while cond { }` and `for x in iterable { }` read like Go's `for`; read `for x in
iter` as `for _, x := range xs` (iterators are Part 7). The Rust-only one is
`loop { }`, an infinite loop; because it's an expression, `break` can carry a
value out:

```rust
let first_even = loop {
    let n = next_number();
    if n % 2 == 0 {
        break n;   // break *with a value* -> becomes the loop's value
    }
};
```

### `return` vs. trailing expression

`return` still exists, for **early** exits:

```rust
fn classify(n: i32) -> &'static str {
    if n < 0 {
        return "negative";   // early return, needs `return` and `;`
    }
    if n == 0 { "zero" } else { "positive" }  // trailing expression = return value
}
```

Idiomatic Rust uses `return` only to bail out early; the normal path falls off
the end as a trailing expression. If a function's last line has no `;` and no
`return`, that line is the result.

> **Trap:** A stray semicolon on the final line silently changes the return type
> to `()`. `fn f() -> i32 { 5; }` does **not** return `5` — the `;` makes it a
> statement and the function returns unit, a compile error against `-> i32`. When
> a "but I clearly returned a value" error confuses you, check for a trailing `;`.

---

## 4. References and dereference — reading level only

You'll see `&`, `&mut`, and `*` constantly. The rules behind them (the borrow
checker) are Part 2; here we only learn to read the symbols.

- `&x` — a shared (read-only) reference to `x`. "Borrow `x`."
- `&mut x` — an exclusive, mutable reference to `x`.
- `*r` — dereference `r` to reach the value it points at.
- `&str`, `&[i32]` in type position — "a reference to a `str` / slice of i32".

```rust
let n = 10;
let r = &n;            // r: &i32
println!("{}", *r);    // prints: 10  (deref to read)

let mut m = 1;
let rm = &mut m;       // exclusive reference
*rm += 1;              // write through it
println!("{m}");       // prints: 2
```

```go
// Go
n := 10
r := &n
fmt.Println(*r) // 10

m := 1
rm := &m
*rm += 1
fmt.Println(m) // 2
```

The surface syntax matches Go almost exactly — `&` takes an address, `*` follows
it — so you can read references right away.

⚡ *Where the Go analogy breaks: in Go `&`/`*` are about pointers and you can have
as many as you like. In Rust `&`/`&mut` are about **borrowing**, and the compiler
enforces a rule with no Go counterpart — many `&` references **or** exactly one
`&mut`, never both, never dangling. The symbols are familiar; the rules they
invoke are the heart of the language (Part 2). Read `&mut` as "I'm temporarily
handing out exclusive write access," not merely "a pointer."*

You'll also see method calls on references without `*` — `r.len()` rather than
`(*r).len()`. That's auto-deref; for reading, just know `.` reaches through
references for you.

---

## 5. The type vocabulary

Annotations go after the name with a colon (`x: i32`), the same name-then-type
order as Go's `x int`.

| Rust | What it is | Closest Go |
| --- | --- | --- |
| `i8 … i64 isize` | signed integers | `int8 … int`; `int` ≈ `isize`/`i64` |
| `u8 … u64 usize` | unsigned integers | `uint8 … uint`; `usize` is the index/len type |
| `f32 f64` | floats | `float32 float64` |
| `bool` | boolean | `bool` |
| `char` | one Unicode scalar (4 bytes) | `rune` |
| `&str` | borrowed string slice | `string` (read-only view) |
| `String` | owned, growable string | `string` (heap-owned, mutable) |
| `(T, U)` | tuple | no first-class equivalent |
| `[T; N]` | fixed-size array of N | `[N]T` |
| `Vec<T>` | growable vector | `[]T` (slice/append) |
| `Option<T>` | a value or nothing | the job of `nil` / `(T, bool)` |

`i32` is the default integer (what a bare literal infers to), like Go's `int`.
`usize` is what indexing and `.len()` use — the pointer-sized unsigned type, so
reach for it when you mean "a size or an index."

### `&str` vs `String`

Go has one `string`. Rust splits the concept: `String` *owns* its bytes on the
heap (growable); `&str` is a *borrowed view* into string data someone else owns —
a literal `"hi"` is a `&'static str`, and `&my_string` coerces to `&str`.

```rust
fn shout(s: &str) -> String {
    s.to_uppercase()        // borrowed view in, new owned String out
}

let owned = String::from("hello");
let view: &str = &owned;           // a borrowed view into it
println!("{}", shout(view));       // prints: HELLO
println!("{}", shout("literal"));  // a &'static str works too
```

```go
// Go
func shout(s string) string {
    return strings.ToUpper(s)
}
```

Rule of thumb: **take `&str` as a parameter, return `String` when you produce new
text.** This is purely an ownership distinction (Part 2); Go folds both roles into
one GC-managed `string`.

### Tuples and arrays

```rust
let point: (i32, i32) = (3, 4);
let x = point.0;          // tuple fields by index: .0, .1, ...
let (px, py) = point;     // or destructure

let trio: [u8; 3] = [1, 2, 3];   // fixed length, part of the type
let zeros = [0u8; 16];           // sixteen zero bytes
```

`[T; N]` is fixed-size — its length is part of its type, like Go's `[16]byte`.
For a growable `[]T`, you want `Vec<T>` (Part 7).

---

## 6. `impl` blocks, methods, and associated functions

Rust has no classes. You define data (a `struct`) and attach behavior in a
separate `impl` block — much as Go separates a type from its methods.

```rust
struct Counter {
    value: i32,
}

impl Counter {
    fn new() -> Counter {          // associated function (no self) — a constructor
        Counter { value: 0 }
    }
    fn increment(&mut self) {      // method: &mut self (mutates)
        self.value += 1;
    }
    fn get(&self) -> i32 {         // method: &self (read-only)
        self.value
    }
}

let mut c = Counter::new();   // associated fn called with `::`
c.increment();                // method called with `.`
println!("{}", c.get());      // prints: 1
```

```go
// Go
type Counter struct{ value int }

func NewCounter() *Counter { return &Counter{} }
func (c *Counter) Increment() { c.value++ } // pointer receiver
func (c *Counter) Get() int   { return c.value }
```

- `&self` ≈ a read-only receiver; `&mut self` ≈ a pointer receiver you mutate
  through. `self` (no `&`) takes ownership — it consumes the receiver, which has
  no Go analog.
- **Methods** (first param is a form of `self`) are called with `.` — `c.get()`.
- **Associated functions** (no `self`) are called with `::` on the type —
  `Counter::new()`. `new` is a convention, not a keyword (≈ Go's `NewCounter`).

The punctuation tells you which you're reading: `Type::thing` is reached *through
the type*; `value.thing` is reached *through a value*.

---

## 7. Paths, `::`, and `use` (full treatment in Part 11)

`::` is the path separator: it walks namespaces — modules, types, associated
functions. Read `std::collections::HashMap` as "the `HashMap` type inside the
`collections` module inside the `std` crate." It's Go's dotted import path plus
the member-access dot, fused into one operator.

```rust
use std::collections::HashMap;   // bring the name into scope (like Go's import)

let mut scores = HashMap::new();         // now usable unqualified
scores.insert("alice", 10);
let n = String::from("hi");              // associated fn via the type path
let path = std::path::Path::new("/tmp"); // or spell the full path inline
```

```go
// Go
import "strings"
s := strings.ToUpper("hi")
```

`use` at the top of a file is Rust's `import`: it pulls a path into scope so you
can write the short name; you can always skip it and fully-qualify inline. For
reading, treat `use X::Y::Z;` as "`Z` now means `X::Y::Z` in this file." Modules
and `pub` are Part 11.

---

## 8. Attributes: `#[derive(...)]`, `#[test]`

Lines starting with `#[...]` are **attributes** — metadata attached to the item
that follows, steering the compiler. The loose Go analogy is struct tags and
build directives (`//go:build`, `` `json:"name"` ``): out-of-band instructions,
not runtime code.

```rust
#[derive(Debug, Clone)]
struct Point {
    x: i32,
    y: i32,
}

let p = Point { x: 1, y: 2 };
let q = p.clone();      // Clone derived above gives us .clone()
println!("{p:?}");      // prints: Point { x: 1, y: 2 }  (Debug derived)
```

`#[derive(...)]` auto-generates trait implementations — here debug formatting and
`.clone()` — so you don't hand-write boilerplate, the way `encoding/json` reads a
struct tag to generate marshalling. You'll also see `#[test]`, marking a function
for `cargo test` (≈ Go's `func TestXxx(t *testing.T)`, but explicit):

```rust
#[test]
fn adds() {
    assert_eq!(add(2, 2), 4);
}
```

When you see `#[...]`, it's an instruction *about* the next item, not part of the
running logic.

---

## 9. Macros and the `!`

`println!`, `vec!`, `format!`, `assert_eq!` — the trailing `!` marks a **macro**
invocation, not a function call. A macro runs at compile time and expands into
code before the program is built. The `!` exists so you can always tell a macro
from a function at a glance.

```rust
let v = vec![1, 2, 3];                 // builds a Vec<i32>
let s = format!("{}-{}", v[0], v[2]);  // returns a String, like Sprintf
println!("{s}");                       // prints: 1-3
```

Why a macro and not a function? `println!` takes a *variable* number of arguments
and type-checks its format string at compile time, neither of which a normal Rust
function can do. Go reaches for variadics and reflection at runtime
(`fmt.Println`, `fmt.Sprintf`); Rust does it at compile time, and the `!` is the
tax. The real mechanics — `macro_rules!`, derive macros, expansion — are the
sibling deep-dive [`../macros.md`](../macros.md).

### Reading format strings

```rust
let name = "Ada";
let n = 42;
println!("{}", name);     // {}   Display: human-facing -> prints: Ada
println!("{:?}", (1, 2)); // {:?} Debug: developer-facing -> prints: (1, 2)
println!("{name} = {n}"); // inline: captures locals by name -> prints: Ada = 42
```

```go
// Go
fmt.Printf("%s\n", name)        // human-facing, like {}
fmt.Printf("%v\n", []int{1, 2}) // value dump, like {:?}
// Go has no inline-variable form; args are always positional
```

`{}` is `%v`/`%s`-ish "display nicely" (the type must implement `Display`); `{:?}`
is "debug-dump" (needs `Debug` — hence `#[derive(Debug)]` above); and `{var}` is
the inline shorthand capturing a local by name. The inline form is the one with
no Go equivalent, and it's what idiomatic modern Rust reaches for.

---

## 10. Comments and doc comments

```rust
// a normal line comment — like Go
/* a block comment */          // also exists; rarely used

/// A doc comment. Documents the *item* that follows.
/// Supports Markdown, rendered by `cargo doc`.
fn answer() -> i32 { 42 }
```

`//` and `/* */` are ordinary comments, identical to Go. The new one is `///`: a
**doc comment** attached to the item below it and consumed by `cargo doc` — the
equivalent of Go's convention where a comment above a declaration becomes its
godoc. The difference is that Rust marks doc comments syntactically (`///`) rather
than by position, and they take Markdown.

---

## One-sentence mental model

Reading Rust is reading Go with three substitutions held in mind: bindings are
immutable unless you see `mut`, almost everything is an expression so a trailing
line with no semicolon is the value, and `!` means "macro" while `::` means
"reach through a type or module" — get those three reflexes and the rest of the
syntax decodes on sight.

---

[← Overview](00-overview.md) · [Next: Ownership and borrowing →](02-ownership-and-borrowing.md)
