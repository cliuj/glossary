# Part 4 — Types and data

[← Lifetimes](03-lifetimes.md) · [Next: Traits and generics →](05-traits-and-generics.md)

The first three parts were about *who owns memory*. This one is about *how you
shape it*. The big mindset shift here is the third one in this guide: in Rust
you design in types first, and you lean on the type system to make illegal
states unrepresentable — the bad case doesn't get a runtime check, it gets a
value that cannot be constructed. A Go developer already builds structs all
day; the new muscle is sum types (`enum`), the death of `nil`, and a compiler
that forces you to handle every case.

---

## 1. Structs: the part that looks like Go

A struct is a named bundle of fields. The surface syntax is so close to Go that
you can almost read it cold:

```rust
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn main() {
    let u = User {
        name: String::from("Ada"),
        age: 36,
        active: true,
    };
    println!("{} is {}", u.name, u.age); // prints: Ada is 36
}
```

```go
// Go
type User struct {
    Name   string
    Age    uint32
    Active bool
}

func main() {
    u := User{Name: "Ada", Age: 36, Active: true}
    fmt.Printf("%s is %d\n", u.Name, u.Age)
}
```

Construction is by named fields, field access is `.`, nesting works the way you
expect. Two differences worth flagging up front. First, there is **no struct
tag** story here: Rust has no `json:"name"` reflection-tag mechanism baked into
the language. Serialization metadata is expressed as *attributes* driven by
crates like `serde` (e.g. `#[serde(rename = "name")]`), which are macros, not a
runtime-reflected string. Second, **field privacy is module-based**, not
casing-based: fields are private to their module unless marked `pub`. There's no
"capital letter means exported" rule — the rules live in the module system,
which is Part 11 (`11-modules-crates-and-cargo.md`).

⚡ *Where the Go analogy breaks: in Go, `Name` is exported and `name` is not, and
that decision is per-identifier by its first letter. In Rust, capitalization is
pure convention (types are `CamelCase`, fields are `snake_case`) and carries no
visibility meaning at all — `pub` does that job, and the default is private.*

There's also a tiny ergonomic win when a variable already has the field's name:

```rust
fn make_user(name: String, age: u32) -> User {
    User { name, age, active: true } // field init shorthand: name == name
}
```

That `name,` is shorthand for `name: name`, the same as Go's struct literal
when you'd otherwise write `Name: name`.

---

## 2. Tuple structs and unit structs

Not every struct needs field names. A **tuple struct** gives you a named type
whose fields are positional:

```rust
struct Point(i32, i32);
struct Meters(f64); // a "newtype" — one field, used for type safety

fn main() {
    let origin = Point(0, 0);
    println!("{} {}", origin.0, origin.1); // access by index: prints 0 0

    let dist = Meters(5.0);
    println!("{}", dist.0); // prints: 5
}
```

The single-field tuple struct (`Meters`) is the **newtype pattern**, and it's
worth its own callout because Go developers reach for it less than they should.
`Meters(f64)` and `Seconds(f64)` are distinct types the compiler will not let
you mix up, even though both wrap an `f64`. Go can do this with `type Meters
float64`, but Go's named types are more permissive about implicit conversion in
arithmetic; Rust's newtype is a hard wall unless you write the conversion.

A **unit struct** has no fields at all:

```rust
struct AlwaysReady; // zero-sized; carries no data

let _ready = AlwaysReady;
```

You use it when you want a type to hang behavior on (an `impl`, §3, or a trait
in Part 5) but there's no data to store. Go's nearest equivalent is
`struct{}` — the zero-sized empty struct you use as a set value or a signal —
but
in Go that's an anonymous shape, whereas `AlwaysReady` is a *named* type you
can implement methods and traits on.

---

## 3. The struct-update syntax

When you want a new struct that's mostly a copy of another with a few fields
changed, use `..other`:

```rust
#[derive(Clone)]
struct Config {
    host: String,
    port: u16,
    tls: bool,
    timeout_ms: u32,
}

fn main() {
    let base = Config {
        host: String::from("localhost"),
        port: 8080,
        tls: false,
        timeout_ms: 3000,
    };

    let secure = Config {
        port: 443,
        tls: true,
        ..base.clone() // take every remaining field from `base`
    };
    println!("{}", secure.host); // prints: localhost
}
```

`..base` fills in `host` and `timeout_ms` from `base`. Go has no syntax for
this — you'd copy the struct and reassign the two fields by hand, or write a
builder. One ownership wrinkle (everything ties back to Part 2): `..base`
*moves* the fields it pulls in, so if any field isn't `Copy`, plain `..base`
consumes `base`. Above I wrote `..base.clone()` to keep `base` usable
afterward; if you don't need `base` again, drop the `.clone()`.

> **Trap:** `..base` is not a spread-and-keep like a JavaScript `...obj`. It
> moves the non-`Copy` fields out of `base`, leaving `base` partially moved and
> unusable as a whole. If you see "borrow of partially moved value," this is
> usually why. Clone the source, or stop using it after the update.

---

## 4. Methods and `impl`: receivers, but enforced

You met `impl` in Part 1; here's the deeper read. Methods live in an `impl`
block, and the receiver — `self` — comes in three flavors that map directly onto
the ownership rules from Part 2:

```rust
struct Counter {
    count: u32,
}

impl Counter {
    // Associated function (no self) — like a Go package-level constructor.
    fn new() -> Counter {
        Counter { count: 0 }
    }

    // &self: borrow immutably. Read-only access, caller keeps ownership.
    fn get(&self) -> u32 {
        self.count
    }

    // &mut self: borrow mutably. Can change fields; needs an exclusive borrow.
    fn bump(&mut self) {
        self.count += 1;
    }

    // self: take ownership. Consumes the value — caller can't use it after.
    fn into_count(self) -> u32 {
        self.count
    }
}

fn main() {
    let mut c = Counter::new(); // associated fn called with ::
    c.bump();
    c.bump();
    println!("{}", c.get());    // prints: 2
    let n = c.into_count();     // c is MOVED here; can't use c afterward
    println!("{n}");            // prints: 2
}
```

The three receivers are exactly the three ways you can touch a value from Part
2: borrow it (`&self`), borrow it exclusively to mutate (`&mut self`), or take
it (`self`). `Counter::new` takes no `self` — it's an **associated function**,
the idiomatic constructor (there's no `new` keyword; it's just a convention).

```go
// Go
type Counter struct{ count uint32 }

func NewCounter() *Counter { return &Counter{} } // constructor by convention

func (c Counter) Get() uint32 { return c.count }  // value receiver  ≈ &self-ish
func (c *Counter) Bump()      { c.count++ }       // pointer receiver ≈ &mut self
```

Go has two receiver kinds — value and pointer — and the choice is yours, with
the compiler mostly indifferent. A value receiver copies; a pointer receiver
shares and can mutate. Rust's `&self` / `&mut self` / `self` maps onto that, but
with two upgrades. First, there are *three* options, because "take ownership and
consume" (`self`) is a distinct, useful thing — that's the `into_*` pattern, a
conversion that destroys the original. Second, the distinction is **enforced**:
if a method takes `&self`, the compiler guarantees it can't mutate; if it takes
`&mut self`, you can only call it on a `mut` binding with an exclusive borrow
available. Go's pointer-vs-value choice is a performance and semantics hint;
Rust's receiver choice is a load-bearing contract.

⚡ *Where the Go analogy breaks: in Go, calling a pointer-receiver method on an
addressable value auto-takes its address, and calling a value-receiver method
silently copies — the receiver kind rarely blocks you. In Rust, calling `bump()`
(`&mut self`) on a non-`mut` binding is a compile error, and calling it while
another borrow is live is a compile error. The receiver isn't a hint; it
participates in the borrow check.*

---

## 5. Enums: the real sum types Go doesn't have

This is the headline feature of Part 4, and the one with no Go equivalent. A
Rust `enum` is a **sum type**: a value that is *exactly one of* several
variants, and each variant can carry its own data of its own shape.

```rust
enum Shape {
    Circle(f64),                         // a radius
    Rect(f64, f64),                      // width, height
    Triangle { base: f64, height: f64 }, // named fields, like a struct variant
    Unit,                                // no data — a C-like bare variant
}

enum Direction { North, East, South, West } // C-like: each is an integer tag
```

One `Shape` value is *a* circle or *a* rect or *a* triangle — never two at once,
never a "rect that accidentally also has a radius field set." The data and the
tag are fused, and (§8) `match` reads the variant back out.

Now the Go contrast, because this is the gap. Go has no sum types, so codebases
fake them two ways, and both leak:

```go
// Go — workaround 1: interface + type switch
type Shape interface{ isShape() }
type Circle struct{ R float64 }
type Rect struct{ W, H float64 }
func (Circle) isShape() {}
func (Rect) isShape() {}
// A type switch over Shape has no exhaustiveness check: add a new implementer
// and every `switch s.(type)` silently falls through to its default.

// Go — workaround 2: a tag field + optional fields (the "struct soup")
type Shape2 struct {
    Kind   string  // "circle" | "rect"
    Radius float64 // only meaningful when Kind == "circle"
    W, H   float64 // only meaningful when Kind == "rect"
}
// Nothing stops you setting Radius AND W on one value. Illegal states are fully
// representable; correctness is a convention you hope holds.
```

Rust's enum closes both holes at once: the set of variants is fixed and known
to the compiler, illegal combinations can't be built, and — as §8 shows —
`match` forces you to handle every variant or the program won't compile.

⚡ *Where the Go analogy breaks: a Go interface is an *open* set — anyone can
implement it, so the compiler can never know all the cases, which is exactly why
type switches can't be checked for exhaustiveness. A Rust enum is a *closed*
set — the variants are declared in one place, so the compiler knows them all and
can demand you handle each one. Open vs. closed is the whole difference.*

---

## 6. `Option<T>`: the death of `nil`

There is no `nil` in Rust. No null pointer, no nil map, no nil slice, no nil
interface. A value of type `User` is *always* a real `User`. When a value might
be absent, you say so in the type with **`Option<T>`**, which is just an enum:

```rust
enum Option<T> {  // this is in the standard library; shown for clarity
    Some(T),
    None,
}
```

So "a user that might not be there" is `Option<User>`, and you physically cannot
use it as a `User` without first dealing with the `None` case:

```rust
fn find_user(id: u32) -> Option<User> {
    if id == 1 {
        Some(User { name: String::from("Ada"), age: 36, active: true })
    } else {
        None
    }
}

fn main() {
    match find_user(1) {
        Some(u) => println!("found {}", u.name), // prints: found Ada
        None => println!("no such user"),
    }
}
```

Here is the contrast that justifies the whole feature. In Go, *every* pointer,
map, slice, interface, channel, and function value can be `nil`, and the type
system does nothing to remind you. Dereferencing a nil pointer or calling a
method on a nil interface panics at runtime — Tony Hoare's "billion-dollar
mistake," shipped in production daily:

```go
// Go
func findUser(id int) *User {
    if id == 1 {
        return &User{Name: "Ada"}
    }
    return nil // the absence is invisible in the type *User
}

func main() {
    u := findUser(2)
    fmt.Println(u.Name) // compiles fine; PANICS at runtime: nil pointer deref
}
```

The Go compiler is perfectly happy with `u.Name` even though `u` is nil. The
absence lives in a value, not in the type, so nothing forces you to check.
Rust's `Option<User>` is a *different type* from `User` — you can't call
`.name` on it, and the compiler stops you at build time. The check isn't
optional vigilance; it's the only way to get at the inner value.

You can extract the value with `.unwrap()` or `.expect()`, but both **panic on
`None`** — they're the explicit "I'm asserting this is present" escape hatch:

```rust
let u = find_user(1).unwrap();             // panics if None
let u = find_user(1).expect("user 1 must exist"); // same, with a message
```

> **Trap:** `.unwrap()` is the `Option`/`Result` equivalent of ignoring an error
> in Go and dereferencing anyway. It's fine in tests, prototypes, and cases you
> can *prove* are present — but every `.unwrap()` in production code is a
> potential panic. The reflex to reach for in real code is `match`, `if let`
> (§8), the `?` operator (Part 6, `06-error-handling.md`), or combinators like
> `.unwrap_or(default)` and `.map(...)`.

Safer access patterns you'll use constantly:

```rust
let name = find_user(2).map(|u| u.name).unwrap_or(String::from("anon"));
// if Some(u), use u.name; if None, fall back to "anon" — no panic
```

⚡ *Where the Go analogy breaks: Go's `nil` is one sentinel shared across many
types, checked (or forgotten) by hand at runtime. `Option<T>` is a real value
with a real `None` variant, checked by the compiler at build time, and it
composes — `Option<Option<T>>` is meaningful, whereas there's no "nil nil." The
cost: absence is now visible everywhere it exists, which feels verbose at first
and turns out to be the point.*

---

## 7. `Result<T, E>`: the `(val, err)` replacement

Where Go returns `(T, error)`, Rust returns one value of type `Result<T, E>` —
again, just an enum:

```rust
enum Result<T, E> { // standard library
    Ok(T),
    Err(E),
}
```

```rust
fn parse_port(s: &str) -> Result<u16, String> {
    match s.parse::<u16>() {
        Ok(n) => Ok(n),
        Err(_) => Err(format!("not a valid port: {s}")),
    }
}

fn main() {
    match parse_port("8080") {
        Ok(p) => println!("port {p}"),       // prints: port 8080
        Err(e) => println!("error: {e}"),
    }
}
```

```go
// Go
func parsePort(s string) (uint16, error) {
    n, err := strconv.ParseUint(s, 10, 16)
    if err != nil {
        return 0, fmt.Errorf("not a valid port: %s", s)
    }
    return uint16(n), nil
}
```

The shapes are cousins, but two differences matter. In Go, success and failure
ride together in a *tuple* `(value, err)` and nothing stops you from using
`value` while `err` is non-nil — the discipline of "check `err` first" is
convention. In Rust, `Result` is *one* value that is **either** `Ok(value)` *or*
`Err(e)`, never both, and you can't get at the `value` without acknowledging the
`Err` case. The full error story — the `?` operator that replaces Go's
`if err != nil` boilerplate, custom error types, and conversion — is Part 6
(`06-error-handling.md`). For now, just see `Result` as a data type: a two-armed
enum, exhaustively handled like any other.

> **Trap:** Rust will warn (and many crates make it a hard error via
> `#[must_use]`) if you call a function returning `Result` and ignore the
> result. That's the opposite of Go, where ignoring a returned `error` with `_`
> or just not assigning it is silent and common. In Rust an unhandled `Result`
> is a smell the compiler points at.

---

## 8. Pattern matching: exhaustive by force

`match` is the tool that makes enums pay off. It's an **expression** (it
produces a value) and it is **exhaustive**: you must cover every possible case,
or the program does not compile.

```rust
enum Direction { North, East, South, West }

fn turn_right(d: Direction) -> Direction {
    match d {
        Direction::North => Direction::East,
        Direction::East => Direction::South,
        Direction::South => Direction::West,
        Direction::West => Direction::North,
        // delete any arm above and you get:
        // ERROR: non-exhaustive patterns: `Direction::West` not covered
    }
}
```

That compile error is the entire selling point. In Go, a `switch` over your
"enum" constants has no exhaustiveness check — forget a case and it silently
falls through, no `default`, no warning (a linter like `exhaustive` can catch
it, but the language doesn't):

```go
// Go
func turnRight(d Direction) Direction {
    switch d {
    case North:
        return East
    case East:
        return South
    // forgot South and West — compiles fine, returns garbage for them
    }
    return d // some "default" you're forced to invent
}
```

`match` also **binds the data** out of enum variants as it matches — this is how
you destructure a `Shape` or pull the `T` out of `Some(T)`:

```rust
fn describe(s: &Shape) -> String {
    match s {
        Shape::Circle(r) => format!("circle r={r}"),       // binds r
        Shape::Rect(w, h) => format!("rect {w}x{h}"),      // binds w, h
        Shape::Triangle { base, .. } => format!("tri b={base}"), // .. = ignore rest
        Shape::Unit => String::from("unit"),
    }
}
```

The rest of the pattern toolbox you'll use often:

```rust
fn classify(n: i32) -> &'static str {
    match n {
        0 => "zero",
        1 | 2 | 3 => "small",          // | is an or-pattern
        4..=9 => "single digit",       // ..= is an inclusive range
        x if x < 0 => "negative",      // `if` is a match guard
        big @ 100..=999 => {           // @ binds the matched value to `big`
            // here `big` is the actual number, and we know it's 100..=999
            let _ = big;
            "three digits"
        }
        _ => "other",                  // _ is the catch-all wildcard
    }
}
```

- `|` — or-patterns: match several patterns in one arm.
- `..=` — inclusive ranges (also `..` exclusive in slice/struct contexts).
- `if <cond>` — a **match guard**: extra runtime condition on an arm.
- `@` — bind the whole matched value to a name *and* test its pattern.
- `_` — wildcard that matches anything and binds nothing (silences
  exhaustiveness — use it deliberately, not to dodge the check).

> **Trap:** `_ => ...` makes any `match` exhaustive instantly, which is
> convenient and sometimes exactly wrong. If you `match` an enum with an
> explicit arm per variant and *no* `_`, then adding a new variant later breaks
> the build at every `match` that needs updating — which is the feature. Slap a
> `_` on and you trade that safety net for silence. Prefer explicit arms for
> your own enums.

---

## 9. `if let`, `while let`, `let ... else`

A full `match` is overkill when you care about exactly one case. `if let` is the
ergonomic single-arm match, and it maps cleanly onto Go's comma-ok idiom:

```rust
let maybe = find_user(1);

if let Some(u) = maybe {
    println!("got {}", u.name); // runs only in the Some case
} else {
    println!("nobody");
}
```

```go
// Go — the comma-ok idiom is the closest cousin
if v, ok := m[key]; ok {
    fmt.Println("got", v.Name) // runs only when the key was present
} else {
    fmt.Println("nobody")
}
```

`if let Some(u) = maybe` reads as "if `maybe` matches `Some(u)`, bind `u` and
run the block" — structurally the same as `if v, ok := m[k]; ok`, except the
binding and the success test are fused into the pattern instead of split across
`v` and `ok`.

`while let` loops as long as a pattern keeps matching — perfect for draining:

```rust
let mut stack = vec![1, 2, 3];
while let Some(top) = stack.pop() { // pop() returns Option<T>; None ends the loop
    println!("{top}"); // prints: 3, then 2, then 1
}
```

`let ... else` handles the "extract or bail" case without nesting. If the
pattern doesn't match, the `else` block runs and must diverge (return, break,
panic):

```rust
fn greet(maybe: Option<User>) {
    let Some(u) = maybe else {
        println!("no user; giving up");
        return; // the else block MUST leave the scope
    };
    // from here down, `u` is a plain User — no nesting, no Option
    println!("hi {}", u.name);
}
```

`let ... else` is the antidote to the rightward-drift you'd otherwise get from
nested `if let`s — it's Rust's version of Go's early-return guard clause
(`if x == nil { return }`), but it also *binds* the unwrapped value for the rest
of the function.

---

## 10. `derive`: free implementations you'll want everywhere

Types you define start with almost no behavior — they can't even be printed or
compared. `#[derive(...)]` is an attribute that auto-generates common trait
implementations so you don't write them by hand:

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let a = Point { x: 1, y: 2 };
    let b = a.clone();              // Clone: a.clone() works
    println!("{a:?}");             // Debug: prints: Point { x: 1, y: 2 }
    println!("{}", a == b);        // PartialEq: prints: true
}
```

The ones you'll reach for constantly:

- **`Debug`** — enables `{:?}` formatting for logging and `dbg!`. Put it on
  basically every type; it's the analog of Go's automatic struct printing with
  `%+v`, except in Rust you opt in (and the opt-in is one word).
- **`Clone`** — gives you `.clone()`, the explicit deep copy from Part 2.
- **`Copy`** — opt into implicit bitwise copy (only allowed if all fields are
  `Copy`); pair it with `Clone`.
- **`PartialEq`/`Eq`** — enable `==`. (Go gives `==` to comparable structs for
  free; Rust makes you ask, so types with intentionally-custom equality don't
  get a wrong default.)
- **`Hash`** — lets the type be a `HashMap`/`HashSet` key.
- **`Default`** — gives `Point::default()` (all fields at their zero/default),
  the closest thing to Go's automatic zero value, except you opt in.

`derive` is itself a macro, and writing your own is the realm of procedural
macros — see the deep-dive `../macros.md` if you want to know what's happening
under the attribute. For now: put `#[derive(Debug, Clone, PartialEq)]` on most
types and add others as the compiler asks.

⚡ *Where the Go analogy breaks: Go gives structs printing (`%+v`), zero values,
and `==` (for comparable types) automatically, with no opt-in. Rust gives you
*nothing* by default and a one-word opt-in for each. It's more typing for a
sharper guarantee: a type without `PartialEq` genuinely can't be compared, so
you never get a surprise default equality on a type where equality is subtle.*

---

## 11. Type aliases, and where this all goes next

A **type alias** gives an existing type a second name. It's a pure synonym — no
new type, no conversion barrier (that's the newtype from §2):

```rust
type UserId = u32;          // UserId IS u32, interchangeable
type Pair = (String, u32);
type ParseResult = Result<u16, String>; // tame a long generic name

fn lookup(id: UserId) -> ParseResult {
    // ...
    Ok(id as u16)
}
```

This is Go's `type UserId = uint32` (the *alias* form with `=`, not the
defined-type form), and it's used the same way: shorten a verbose type or
document intent without changing behavior. For a hard boundary the compiler
enforces, you want the newtype `struct UserId(u32)` from §2, not an alias.

Finally, the forward pointer: you've now met `Option<T>` and `Result<T, E>`,
both **generic enums** — enums parameterized by a type. That `<T>` is the same
machinery that powers `Vec<T>`, `HashMap<K, V>`, and every container in the
standard library, and it's how you write code that works over many types without
giving up type safety. Generics and the traits that constrain them are Part 5
(`05-traits-and-generics.md`) — which is also where `derive`'s traits stop being
magic and become things you can define and implement yourself.

---

## One-sentence mental model

Design your data as structs for "all of these fields together" and enums for
"exactly one of these cases," let `Option<T>` and `Result<T, E>` carry absence
and failure *in the type* so there's no `nil` and no unchecked `(val, err)`, and
lean on exhaustive `match` to make the compiler prove you've handled every case
Go would have let you forget.

---

[← Lifetimes](03-lifetimes.md) · [Next: Traits and generics →](05-traits-and-generics.md)
