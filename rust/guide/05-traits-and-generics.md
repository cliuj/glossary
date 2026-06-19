# Part 5 — Traits and generics

[← Types and data](04-types-and-data.md) · [Next: Error handling →](06-error-handling.md)

This is the part where your Go instincts are the most help and the most danger
at the same time. Rust's traits *are* the interface idea you already know — a
named set of method signatures a type can satisfy — and for a while you can read
trait code as "Go interfaces with different spelling." Then four things diverge:
implementation is explicit, you can implement traits for types you don't own,
generics are monomorphized instead of boxed, and "interface value" is a separate
thing called a trait object. Get those four and you've got traits.

---

## 1. A trait is an interface (mostly)

Start with the part that maps cleanly. A trait declares method signatures; a
type *implements* the trait by providing bodies for them.

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Article {
    title: String,
    body: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}...", self.title, &self.body[..20])
    }
}

fn main() {
    let a = Article {
        title: String::from("Rust"),
        body: String::from("Traits are not interfaces, except when they are."),
    };
    println!("{}", a.summarize()); // prints: Rust: Traits are not int...
}
```

```go
// Go
type Summary interface {
    Summarize() string
}

type Article struct {
    Title string
    Body  string
}

func (a Article) Summarize() string {
    return fmt.Sprintf("%s: %s...", a.Title, a.Body[:20])
}
```

The shapes line up: `trait` ≈ `interface`, `impl Summary for Article` ≈ "Article
has the methods Summary wants." The `&self` receiver is the method's receiver,
exactly like `(a Article)` in Go (receivers are Part 4,
`04-types-and-data.md`). If you stopped reading here you'd be 70% productive.
The rest of this file is the 30% that bites.

---

## 2. Implementation is explicit, not structural

In Go, a type satisfies an interface *implicitly*: if `Article` happens to
have a `Summarize() string` method, it satisfies `Summary`, and nobody said so.
The relationship is structural and discovered by the compiler.

Rust is the opposite. A type implements a trait only when you write an explicit
`impl Trait for Type` block. Having a method with the right name and signature
is not enough — without the `impl`, the type does *not* implement the trait.

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Article { title: String }

// This method exists, but it's NOT an impl of Summary.
impl Article {
    fn summarize(&self) -> String {
        self.title.clone()
    }
}

fn print_it(s: &impl Summary) {
    println!("{}", s.summarize());
}

fn main() {
    let a = Article { title: String::from("hi") };
    print_it(&a);
    // ERROR: the trait bound `Article: Summary` is not satisfied
}
```

```go
// Go — no impl block exists or is needed. Having the method IS satisfying
// the interface; the compiler discovers it structurally at the call site.
type Article struct{ Title string }

func (a Article) Summarize() string { return a.Title }

func printIt(s Summary) { fmt.Println(s.Summarize()) }

func main() {
    printIt(Article{Title: "hi"}) // compiles — Article structurally fits Summary
}
```

That's a feature, not a bureaucratic tax. Two consequences a Go dev feels right
away:

- **No accidental satisfaction.** In Go a refactor that renames a method can
  silently make a type stop satisfying an interface, with the failure surfacing
  far away. In Rust the `impl` is a declared, greppable contract. Intent is
  explicit.
- **A type can implement two traits with the same method name** and stay
  unambiguous, because you opt into each trait by name. Go's single method set
  has no such distinction.

⚡ *Where the Go analogy breaks: there is no "implements" relationship floating
in the ether. `impl Trait for Type` is a thing you write, the compiler checks,
and other code can rely on. Structural typing is gone; nominal, declared
conformance is in.*

> **Trap:** to *call* a trait's methods on a value, the trait must be in scope.
> If `summarize` "doesn't exist" on a type you know implements `Summary`, you're
> usually missing a `use some_crate::Summary;`. The method is defined by the
> trait, so the trait has to be imported, even though the type isn't.

---

## 3. The orphan rule, and implementing your trait for `i32`

Here's something Go simply cannot do. You can implement *your own* trait for a
type you didn't define — including standard-library types like `i32`, `String`,
or `Vec<T>`.

```rust
trait Doubled {
    fn doubled(&self) -> Self;
}

impl Doubled for i32 {
    fn doubled(&self) -> i32 {
        self * 2
    }
}

fn main() {
    println!("{}", 21.doubled()); // prints: 42
}
```

In Go you cannot declare a method on `int` — methods can only be defined on
types in the same package, so extending a foreign type means wrapping it in a
new named type. Rust lets you bolt behavior straight onto the existing type,
which is how crates teach `i32` and `String` new tricks without wrappers.

The catch is the **coherence / orphan rule**, and it's simpler than it sounds.
For any `impl Trait for Type`, *you must own at least one of the two* — either
the trait is defined in your crate, or the type is. You cannot write
`impl Display for Vec<T>` because both `Display` and `Vec` belong to other
crates; that pairing isn't yours to define.

```rust
use std::fmt;

// ERROR: only traits defined in the current crate can be implemented
//        for types defined outside of the crate (orphan rule)
impl fmt::Display for Vec<i32> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:?}", self)
    }
}
```

Why the rule exists: coherence guarantees there is *exactly one* implementation
of a given trait for a given type across the entire program. If two unrelated
crates could both `impl Display for Vec<i32>`, linking them would be ambiguous —
which impl wins? The orphan rule makes that conflict impossible by construction.
Go dodges this whole problem because it never lets you attach methods to foreign
types in the first place.

> **Trap:** the standard workaround when you need an impl you don't own is the
> **newtype pattern** — wrap the foreign type in a one-field tuple struct you
> *do* own (`struct Meters(f64);`) and implement the trait on the wrapper. You
> already met newtypes in Part 4 (`04-types-and-data.md`); this is their other
> main job.

---

## 4. Default methods and trait bounds on methods

A trait can supply a *default* body. Implementors get it for free and may
override it — this is the closest Rust has to interface composition with shared
behavior.

```rust
trait Summary {
    fn title(&self) -> String;

    // Default method, written in terms of another trait method.
    fn summarize(&self) -> String {
        format!("(read more about {}...)", self.title())
    }
}

struct Article { headline: String }

impl Summary for Article {
    fn title(&self) -> String {
        self.headline.clone()
    }
    // summarize() not written: the default is used.
}

fn main() {
    let a = Article { headline: String::from("Rust 2.0") };
    println!("{}", a.summarize()); // prints: (read more about Rust 2.0...)
}
```

Go interfaces have no default methods at all — every method in an interface is
abstract, and shared default behavior is done by embedding a struct, not the
interface. Rust folds the default into the trait itself.

A trait can also require its implementors to *already* implement another trait
(a supertrait), and individual methods can carry their own bounds. We'll see the
bound syntax properly in §6; the point here is that "this method only exists
when `T: Ord`" is expressible.

---

## 5. Generics and monomorphization

A generic function is parameterized by a type, written `<T>`, with the
constraints listed as **trait bounds**. Here's the canonical example — largest
element of a slice — which requires that elements be comparable (`PartialOrd`)
and copyable (`Copy`):

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut biggest = list[0];
    for &item in list {
        if item > biggest {
            biggest = item;
        }
    }
    biggest
}

fn main() {
    let nums = [3, 7, 2, 9, 4];
    println!("{}", largest(&nums)); // prints: 9
}
```

```go
// Go (generics landed in 1.18)
func Largest[T cmp.Ordered](list []T) T {
    biggest := list[0]
    for _, item := range list {
        if item > biggest {
            biggest = item
        }
    }
    return biggest
}
```

This looks almost identical, and conceptually it is: both languages let you
write one function over many types with the constraints spelled out. The
interesting divergence is *how the compiler makes it real*.

Rust uses **monomorphization**. For every concrete type you actually call
`largest` with, the compiler stamps out a specialized copy — `largest::<i32>`,
`largest::<f64>`, and so on — each compiled as if you'd hand-written it for that
type. There is no boxing, no indirection, no runtime type information. That
`largest::<i32>` is as fast as a hand-written `i32` version. The cost is paid
in compile time and binary size (more code generated), not at runtime.

Go's generics use a different strategy: **GCShape stenciling and dictionaries**.
Roughly, Go generates one copy of the function per memory-layout "shape" (so all
pointer-shaped types share an instantiation) and passes a hidden *dictionary*
argument carrying the per-type information. That keeps binaries small but
introduces indirection — for many types the operations go through the dictionary
rather than being inlined to a direct call. It's a deliberate tradeoff favoring
compile speed and code size over raw runtime specialization.

⚡ *Where the Go analogy breaks: Rust's generics predate Go's by years and reach
further — associated types (§8), generic trait impls, const generics, and
specialization-adjacent patterns. But the headline difference is the dispatch
model: Rust generics are zero-cost and monomorphized; Go generics carry runtime
machinery. If you want Go-style "one shared implementation behind a pointer," in
Rust you reach for a trait object (§7), not a generic.*

For longer bound lists, the inline `T: A + B + C` syntax gets noisy. The `where`
clause moves bounds below the signature; it's purely cosmetic but standard for
anything non-trivial:

```rust
fn process<T, U>(a: T, b: U) -> String
where
    T: std::fmt::Display + Clone,
    U: std::fmt::Debug,
{
    format!("{} / {:?}", a, b)
}
```

---

## 6. Trait bounds as the unit of abstraction

The mental model that pays off: in Rust you rarely abstract over "a type," you
abstract over "a type that can do these things." The bound list *is* the
abstraction. `T: Clone + Debug` means "any type I can clone and format for
debugging," and that's all this code is allowed to assume about `T`.

For arguments and return values there's a lighter syntax, `impl Trait`:

```rust
use std::fmt::Display;

// Argument position: "some type that implements Display."
// This is sugar for fn show<T: Display>(item: T).
fn show(item: impl Display) {
    println!("{}", item);
}

// Return position: "I return SOME concrete type implementing Iterator,
// and the caller doesn't get to know which."
fn counter() -> impl Iterator<Item = u32> {
    (0..3).map(|x| x * 10)
}

fn main() {
    show("hello");                       // prints: hello
    let v: Vec<u32> = counter().collect();
    println!("{:?}", v);                 // prints: [0, 10, 20]
}
```

`impl Trait` in return position is genuinely useful and has no Go analog: it
lets you return a complex concrete type (a chained iterator whose real type is
unspeakably long) while exposing only the trait. The function is still
monomorphized and the return is still one concrete type — you've just hidden its
name. Contrast Go, where returning an interface value is the *only* way to
return "something that implements I," and it always boxes.

> **Trap:** `impl Trait` in return position promises *one single* concrete type.
> You can't have two `return` branches yielding different concrete types behind
> one `impl Iterator` — that's `// ERROR: ... if and else have incompatible
> types`. When you genuinely need to return different concrete types from
> different branches, you need a trait object (§7), e.g. `Box<dyn Iterator>`.

---

## 7. Static vs dynamic dispatch — this is where Go interfaces live

This is the most important section for a Go developer, so slow down here.

Everything in §5–§6 was **static dispatch**: generics are monomorphized, the
exact method to call is known at compile time, and the call is a direct,
inlinable jump. Zero runtime cost, but the function gets duplicated per type and
every concrete type is fixed at compile time.

**Dynamic dispatch** is the other mode, and it is *exactly* what a Go interface
value is. A **trait object** — written `dyn Trait` — is a fat pointer: one
pointer to the data, one pointer to a **vtable** of that type's method
implementations. The concrete type is erased; method calls go through the vtable
at runtime. You hold it behind a pointer, almost always `Box<dyn Trait>` (owned)
or `&dyn Trait` (borrowed).

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Article { title: String }
struct Tweet { user: String }

impl Summary for Article {
    fn summarize(&self) -> String { format!("article: {}", self.title) }
}
impl Summary for Tweet {
    fn summarize(&self) -> String { format!("@{}", self.user) }
}

fn main() {
    // A Vec of DIFFERENT concrete types, unified by the trait object.
    let items: Vec<Box<dyn Summary>> = vec![
        Box::new(Article { title: String::from("Rust") }),
        Box::new(Tweet { user: String::from("ferris") }),
    ];
    for item in &items {
        println!("{}", item.summarize());
    }
    // prints: article: Rust
    // prints: @ferris
}

// Borrowed trait object as a parameter — dispatched at runtime.
fn announce(s: &dyn Summary) {
    println!("Breaking: {}", s.summarize());
}
```

```go
// Go — a []Summary is, under the hood, exactly Vec<Box<dyn Summary>>.
type Summary interface {
    Summarize() string
}

func main() {
    items := []Summary{
        Article{Title: "Rust"},
        Tweet{User: "ferris"},
    }
    for _, item := range items {
        fmt.Println(item.Summarize())
    }
}

func Announce(s Summary) { // s is a fat pointer: (data, itable)
    fmt.Println("Breaking:", s.Summarize())
}
```

Stare at those two. A Go `Summary` value is a two-word pair: a pointer to the
data and a pointer to an *itable* (interface method table). Rust's `dyn Summary`
behind a pointer is the same two-word pair: data pointer plus vtable pointer.
**Go interfaces are trait objects with the dynamic-dispatch decision made for
you, always.** In Rust you choose per use site.

How to choose:

- **Reach for generics / `impl Trait` (static)** by default. It's the zero-cost
  path, it inlines, and most code abstracts over one type per call site anyway.
- **Reach for `dyn Trait` (dynamic)** when you need a *heterogeneous* collection
  (a `Vec` holding several different concrete types, like above), when you want
  to avoid code bloat from many monomorphized copies, or when the concrete type
  must be chosen at runtime (e.g. plugin-style dispatch). This is precisely the
  situation where a Go dev instinctively reaches for `[]Interface`.

⚡ *Where the Go analogy breaks: in Go every interface call is a dynamic vtable
call — you pay the indirection whether you need it or not. In Rust dynamic
dispatch is opt-in (`dyn`), and the default (generics) is statically dispatched
and free. The Go cost is the Rust opt-in.*

> **Trap:** not every trait can be a trait object. A trait is **object-safe**
> only if its methods don't return `Self` and aren't generic over a type, among
> other rules (a generic method can't go in a vtable — there's no single body to
> point at). If you see `// ERROR: the trait ... cannot be made into an object`,
> that's why: the trait works fine with generics but can't be `dyn`-ed.

---

## 8. Associated types vs generic parameters

A trait can declare an **associated type** — a type the implementor fills in,
written `type Name;`. The standard example you'll meet constantly is `Iterator`:

```rust
trait Iterator {
    type Item;                       // associated type, filled in per impl
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter { n: u32 }

impl Iterator for Counter {
    type Item = u32;                 // this iterator yields u32s
    fn next(&mut self) -> Option<u32> {
        if self.n < 3 {
            self.n += 1;
            Some(self.n)
        } else {
            None
        }
    }
}
```

Why an associated type instead of a generic parameter like
`trait Iterator<Item>`? Because there is **exactly one** sensible `Item` per
implementing type — a `Counter` always yields `u32`, never several choices. An
associated type encodes "this implementor pins down exactly one related type,"
whereas a generic parameter would say "you can implement this trait many times
with many different `Item`s for the same type," which is wrong here and would
force callers to annotate the type everywhere.

Rule of thumb: **one related type per impl → associated type. Many independent
implementations → generic parameter.** Go has no equivalent because Go
interfaces can't carry associated types at all; the closest Go gets is a generic
constraint interface, but that ties the type to the *function's* type
parameters, not to the implementing type. `Iterator` is Part 7
(`07-collections-iterators-closures.md`).

---

## 9. The standard traits you'll meet on day one

You don't have to define all your traits — the standard library ships a small
set you'll implement (usually via `#[derive(...)]`, which you saw in Part 4) and
rely on constantly.

```rust
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Default, Hash)]
struct Point { x: i32, y: i32 }
```

What each one buys you, with the Go reflex it replaces:

- **`Debug`** — the `{:?}` formatter, for developer/debug output. This is what
  `#[derive(Debug)]` gives you. Closest Go reflex: `%+v` / `%#v`. Almost every
  type should derive it.
- **`Display`** — the `{}` formatter, for user-facing output. *Not* derivable;
  you write it by hand (`impl fmt::Display`). Go analog: implementing
  `fmt.Stringer`'s `String() string`.
- **`Clone`** — explicit deep copy via `.clone()`. There's no implicit copying
  of owned data in Rust (ownership is Part 2, `02-ownership-and-borrowing.md`).
  Go copies structs implicitly on assignment; Rust makes you ask.
- **`Copy`** — a *marker* trait (no methods) meaning "this type is cheap, plain
  bits, and assigning it copies instead of moves." Only for small POD-like types
  (`i32`, `bool`, small structs of `Copy` fields). Go has no notion of move vs
  copy, so this distinction is new.
- **`PartialEq` / `Eq`** — `==`. `PartialEq` is the actual `==`; `Eq` is a
  marker promising the relation is *total* (no `f64`-style `NaN != NaN` case —
  floats are `PartialEq` but not `Eq`). Go's `==` is built in and roughly
  matches `PartialEq`.
- **`PartialOrd` / `Ord`** — `<`, `>`, sorting. `Ord` is required by
  `.sort()` and `BTreeMap` keys. Go uses `cmp.Ordered` / custom `Less` funcs.
- **`Default`** — `T::default()`, the zero-ish value. Go's zero value is
  automatic and language-level; Rust's `Default` is an explicit, derivable
  trait you opt into.
- **`From` / `Into`** — value conversions. Implement `From<A> for B` and you get
  `Into` for free (see blanket impls, §10). This is the machinery the `?`
  operator uses to convert error types automatically — covered in Part 6
  (`06-error-handling.md`).
- **`Iterator`** — anything you can loop over with `for`; Part 7
  (`07-collections-iterators-closures.md`).
- **`Send` / `Sync`** — marker traits about thread-safety; the compiler usually
  derives them for you. They're how Rust enforces data-race freedom at compile
  time, and they're Part 9 (`09-concurrency.md`).

Operators are traits too. `+` is `std::ops::Add`, `*` is `Mul`, indexing is
`Index`, and so on — implement the trait and the operator works on your type.
Go has no operator overloading at all, so this is genuinely new; use it
sparingly and only where the math meaning is obvious.

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct V2 { x: f64, y: f64 }

impl Add for V2 {
    type Output = V2;                         // an associated type again
    fn add(self, rhs: V2) -> V2 {
        V2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}

fn main() {
    let sum = V2 { x: 1.0, y: 2.0 } + V2 { x: 3.0, y: 4.0 };
    println!("{:?}", sum); // prints: V2 { x: 4.0, y: 6.0 }
}
```

---

## 10. Blanket impls

A **blanket impl** is an `impl` written for *every* type that satisfies some
bound, rather than for one concrete type. The standard library uses this to give
you `Into` for free: there is roughly an `impl<T, U> Into<U> for T where U:
From<T>` in `std`, so implementing `From` automatically yields the matching
`Into` for all types — you never write `Into` by hand.

```rust
struct Celsius(f64);
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

fn main() {
    let f: Fahrenheit = Celsius(100.0).into(); // .into() came free from From
    println!("{}", f.0); // prints: 212
}
```

Go has nothing like this — you cannot say "every type satisfying X also gets
method set Y" in one declaration. It's a direct consequence of the explicit,
coherent impl system from §2–§3: because impls are first-class and globally
coherent, the library can write one that applies across all qualifying types.

---

## One-sentence mental model

A trait is a Go interface you must implement *explicitly* and may implement even
for types you don't own (within the coherence rule); reach for it as a generic
bound for zero-cost monomorphized static dispatch by default, and as a
`dyn Trait` trait object — the exact thing a Go interface value already is —
when you need runtime dispatch or a heterogeneous collection.

---

[← Types and data](04-types-and-data.md) · [Next: Error handling →](06-error-handling.md)
