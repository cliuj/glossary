# Part 7 — Collections, iterators, closures

[← Error handling](06-error-handling.md) · [Next: Smart pointers and interior mutability →](08-smart-pointers-and-interior-mutability.md)

You already have slices, maps, and `for` loops in your head from Go. The data
structures port over almost one-for-one, so the surface area is small. The real
content of this part is **iterators**: lazy, chainable pipelines that replace
most of the hand-written `for` loops you write in Go, compile down to the same
machine code, and lean hard on the ownership rules from Part 2
(`02-ownership-and-borrowing.md`). Closures come along for the ride, because
they're what you feed those pipelines.

---

## 1. `Vec<T>`: the growable slice

`Vec<T>` is the workhorse owned, growable, heap-allocated sequence — the
closest thing to a Go slice. You `push`, you index, it grows.

```rust
fn main() {
    let mut v: Vec<i32> = Vec::new();
    v.push(10);
    v.push(20);
    v.push(30);

    let v2 = vec![1, 2, 3]; // the vec! macro: literal construction

    println!("{}", v[0]);     // indexing: prints 10
    println!("{}", v.len());  // prints: 3
    let last = v.pop();       // pop() returns Option<i32>: Some(30)
    println!("{last:?}");     // prints: Some(30)
}
```

```go
// Go
func main() {
    v := []int{}
    v = append(v, 10, 20, 30)
    v2 := []int{1, 2, 3}
    _ = v2
    fmt.Println(v[0])    // 10
    fmt.Println(len(v))  // 3
    last := v[len(v)-1]  // no built-in pop; slice it yourself
    v = v[:len(v)-1]
    fmt.Println(last)
}
```

The mechanics line up: both own a heap buffer with a length and a capacity, and
both grow by reallocating when capacity runs out. The differences are the usual
ownership ones. A `Vec` has a single owner; passing it by value *moves* it. To
let a function read without taking it, you pass a borrow — and that borrow is a
**slice**, `&[T]`, which is §3.

The indexing gap is worth its own callout. `v[i]` panics on an out-of-bounds
index, exactly like Go panics on `v[i]` past the length. But Rust also gives you
the checked form, `.get(i)`, which returns `Option<&T>` instead of panicking:

```rust
let v = vec![1, 2, 3];
let x = v[5];            // PANICS: index out of bounds
let y = v.get(5);        // None — no panic
let z = v.get(1);        // Some(&2)
println!("{y:?} {z:?}"); // prints: None Some(2)
```

> **Trap:** `v[i]` and `v.get(i)` are not interchangeable. `v[i]` gives you a
> `T` (well, a place you can borrow or copy out of) and panics on a bad index;
> `v.get(i)` gives you an `Option<&T>` and never panics. Reach for `.get` at any
> boundary where the index came from outside your control — it's the same
> instinct that makes you check `i < len(v)` in Go, except the `Option` makes
> forgetting impossible.

⚡ *Where the Go analogy breaks: Go slices are cheap views that share an
underlying array — `s[1:3]` aliases `s`, and three slices can point into one
backing array with no ownership conflict, because the GC keeps the array alive.
A `Vec<T>` is the *owner* of its buffer, not a view. The view type is the
separate `&[T]` slice, and the borrow checker forbids handing out a `&mut`
slice while any other slice into the same `Vec` is live. Go's "slices alias
freely" simply isn't a thing a `Vec` does.*

---

## 2. `HashMap<K, V>`: maps without the nil

`HashMap<K, V>` is Go's `map[K]V`. Insert with `.insert`, look up with `.get`.
The lookup is where the design diverges:

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<String, i32> = HashMap::new();
    scores.insert(String::from("ada"), 10);
    scores.insert(String::from("bob"), 7);

    // .get returns Option<&V> — None if the key is absent, no nil, no zero value.
    match scores.get("ada") {
        Some(s) => println!("ada has {s}"), // prints: ada has 10
        None => println!("no ada"),
    }

    // .get on a missing key is None, not a silent zero:
    println!("{:?}", scores.get("nobody")); // prints: None
}
```

```go
// Go
func main() {
    scores := map[string]int{}
    scores["ada"] = 10
    scores["bob"] = 7

    if s, ok := scores["ada"]; ok { // the comma-ok idiom
        fmt.Println("ada has", s)
    }
    fmt.Println(scores["nobody"]) // prints: 0 — the zero value, silently
}
```

Go's `m[key]` returns the value's zero value for a missing key, and you opt into
knowing-it-was-missing with the comma-ok form. Rust has no zero value, so
`.get` *must* return `Option<&V>` — there's no fallback `V` to hand back. The
`Some`/`None` is the comma-ok `ok` fused into the return value, the same trade
you saw with `Option` in Part 4 (`04-types-and-data.md`).

The piece with no clean Go equivalent is the **`entry` API**, for the
"get-or-insert, then update" pattern. In Go you write the double lookup by hand;
in Rust one call gives you a mutable handle into the slot:

```rust
use std::collections::HashMap;

fn main() {
    let text = "the cat the dog the bird";
    let mut counts: HashMap<&str, i32> = HashMap::new();

    for word in text.split_whitespace() {
        // entry(word): a handle to the slot. or_insert(0): default if absent.
        // Returns &mut i32 either way, so we can += in place.
        *counts.entry(word).or_insert(0) += 1;
    }

    println!("{}", counts["the"]); // prints: 3
}
```

```go
// Go — the hand-written two-step
counts := map[string]int{}
for _, word := range strings.Fields(text) {
    counts[word]++ // Go's map index auto-zeroes, so ++ "just works" here
}
```

Go's `counts[word]++` quietly relies on the zero value to seed the counter — one
of the few places the zero value is a genuine ergonomic win. Rust makes the
default explicit with `.or_insert(0)`, but in exchange `entry` generalizes: when
the default is expensive, `.or_insert_with(|| build())` runs the closure only on
a miss, and `.and_modify(|v| ...)` lets you update an existing entry without a
second lookup. It's one hash probe, not two.

> **Trap:** `counts["the"]` (indexing a `HashMap`) **panics** if the key is
> absent, just like a `Vec` index. It's not the Go "give me the zero value"
> index. For anything but a key you've proven is present, use `.get(k)` and
> handle the `Option`.

The other standard collections, briefly: `BTreeMap<K, V>` is a sorted map
(iterates in key order, like nothing in Go's stdlib without sorting the keys
yourself); `HashSet<T>` / `BTreeSet<T>` are sets (Go fakes these with
`map[T]struct{}`); and `VecDeque<T>` is a double-ended queue for cheap
push/pop at both ends. They all share the iterator machinery below.

---

## 3. `&str`, `String`, and `&[T]`: the owned/borrowed split

You met `String` vs `&str` back in Part 1, but it's worth re-seating it next to
the collection types, because the same owned-vs-borrowed split runs through all
of them:

| Owned (heap, you own it) | Borrowed (a view, you don't) |
| ------------------------ | ---------------------------- |
| `String`                 | `&str`                       |
| `Vec<T>`                 | `&[T]`                       |

```rust
fn first_word(s: &str) -> &str {        // takes a borrowed view
    s.split_whitespace().next().unwrap_or("")
}

fn sum(xs: &[i32]) -> i32 {             // &[i32], not &Vec<i32>
    let mut total = 0;
    for x in xs { total += x; }
    total
}

fn main() {
    let owned: String = String::from("hello world");
    let v: Vec<i32> = vec![1, 2, 3];

    // A String coerces to &str, and a Vec to &[T], automatically at call sites:
    println!("{}", first_word(&owned)); // prints: hello
    println!("{}", sum(&v));            // prints: 6
}
```

The rule of thumb that mirrors Go practice: **accept the borrowed form**
(`&str`, `&[T]`) in function parameters, and return or store the owned form
(`String`, `Vec<T>`) when you're handing back something the caller must own.
Taking `&str` instead of `String` is the Rust analog of taking `[]byte` or a
`string` value in Go rather than demanding the caller surrender ownership — it
makes your function callable from the widest set of sources (string literals,
slices of a bigger `String`, etc.).

⚡ *Where the Go analogy breaks: Go has one `string` type (immutable, a header
over bytes) and one `[]T` slice type, and "owned vs borrowed" isn't a
distinction the language draws — the GC sorts out lifetimes. Rust splits each
into an owning type and a borrowing type precisely because there's no GC: the
borrow's lifetime must be provably shorter than the owner's, which is the
Part 3 (`03-lifetimes.md`) machinery quietly at work.*

---

## 4. The `Iterator` trait, and what `iter` yields

Here's the headline. An **iterator** is any type implementing the `Iterator`
trait, which is essentially one method:

```rust
pub trait Iterator {
    type Item;                          // associated type: what it yields (Part 5)
    fn next(&mut self) -> Option<Self::Item>; // Some(x) for each item, then None
}
```

That `type Item` is an **associated type** — the Part 5
(`05-traits-and-generics.md`) feature — and `next` returning `Option` is how the
iterator signals "I'm done" (`None`) versus "here's the next one" (`Some`). You
rarely call `next` directly; you build pipelines on top of it.

You get an iterator from a collection three ways, and *which one you pick
decides what you can do with the items* — because it's pure Part 2 ownership:

```rust
fn main() {
    let v = vec![String::from("a"), String::from("b")];

    // iter()      yields &T       — borrow each item, collection untouched
    for s in v.iter() {
        println!("{s}"); // s: &String
    }
    println!("{}", v.len()); // still usable: prints 2

    let mut v2 = vec![1, 2, 3];
    // iter_mut()  yields &mut T   — mutably borrow each item, edit in place
    for n in v2.iter_mut() {
        *n *= 10;
    }
    println!("{v2:?}"); // prints: [10, 20, 30]

    // into_iter() yields T        — MOVES each item out; consumes the collection
    for s in v.into_iter() {
        println!("owns {s}"); // s: String, owned
    }
    // v is gone here — into_iter consumed it.
}
```

The three methods are the three ways to touch a value, applied per-element:
`iter()` borrows (`&T`), `iter_mut()` borrows exclusively (`&mut T`),
`into_iter()` takes ownership (`T`). Memorize this table — it's the single most
common source of "why won't this compile" once you're past the basics.

⚡ *Where the Go analogy breaks: `for i, x := range v` in Go always hands you a
**copy** of each element, and you mutate the original by writing back through
`v[i]`. Rust forces you to *say* which access you want up front: a read-only
borrow, a mutable borrow, or ownership. `range` is one fixed behavior; `iter` /
`iter_mut` / `into_iter` are three deliberate ones, and the borrow checker holds
you to your choice for the duration of the loop.*

> **Trap:** `into_iter()` **consumes** the collection. After
> `for x in v.into_iter()` (or the bare `for x in v`, which is the same thing —
> see §6), `v` is moved and you can't use it again. If you need the collection
> afterward, use `v.iter()` (borrow) instead. The compiler error is "use of
> moved value: `v`," and this is the usual cause.

---

## 5. Laziness: adapters build, consumers run

This is the section that changes how you write code. Iterator methods come in
two kinds:

- **Adapters** (`map`, `filter`, `take`, `skip`, `zip`, `enumerate`, `chain`,
  `flat_map`, `rev`, ...) return *another iterator*. They do **no work**. They
  just describe a transformation, lazily.
- **Consumers** (`collect`, `sum`, `count`, `fold`, `reduce`, `find`, `any`,
  `all`, `min`, `max`, and the `for` loop) actually *drive* the iterator,
  pulling items through the whole adapter chain one at a time.

Nothing happens until a consumer runs. The chain is a recipe; the consumer
cooks it.

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6];

    // Adapters only — this line does NOTHING and the compiler warns about it:
    let _lazy = nums.iter().map(|n| n * 2).filter(|n| n % 3 == 0);
    // WARNING: unused `Map`/`Filter` that must be used (iterators are lazy)

    // Add a consumer and it runs, top to bottom, one item at a time:
    let doubled_div3: Vec<i32> = nums
        .iter()
        .map(|n| n * 2)         // 2 4 6 8 10 12
        .filter(|n| n % 3 == 0) // keep multiples of 3: 6 12
        .collect();             // <-- the consumer
    println!("{doubled_div3:?}"); // prints: [6, 12]
}
```

Now the comparison that's the whole reason this part exists. Go has **no
standard iterator-chaining idiom**. You write the loop by hand:

```go
// Go — the same computation, hand-written
nums := []int{1, 2, 3, 4, 5, 6}
var result []int
for _, n := range nums {
    d := n * 2
    if d%3 == 0 {
        result = append(result, d)
    }
}
// result == [6 12]
```

Both compile to roughly the same instructions (see §10 — Rust's chain is
zero-cost). The difference is expression. The Rust version reads as a sequence
of transforms — *double, then keep multiples of 3, then collect* — with no
mutable accumulator and no index bookkeeping. The Go version is a loop with a
manually-managed `result` slice and nested conditionals; correct, but you have
to *read the body* to learn what it computes. For a one-off that's fine; across
a codebase, the iterator chain is the reason Rust code has dramatically fewer
hand-rolled loops than Go.

Go 1.23 added range-over-func (`for x := range seq`) and an `iter` package, so
the *capability* now exists, and libraries are starting to expose iterators. But
the **culture** is still hand-written `for` loops — the stdlib has no `Map` /
`Filter` / `Collect`, and idiomatic Go leans on explicit loops. In Rust, the
chain *is* the idiom.

> **Trap:** Forgetting the consumer is the rookie iterator bug.
> `v.iter().map(f)` on its own runs `f` exactly **zero** times — it builds a
> `Map` struct and
> drops it. If your `map` / `filter` side effects never fire, you forgot to
> `collect`, `for`, `sum`, or otherwise consume. The good news: the compiler
> emits a `must_use` warning for a dropped iterator, so it usually catches you.

⚡ *Where the Go analogy breaks: a Go `for` loop is eager and concrete — the body
runs now, in place. A Rust adapter chain is a lazily-composed *value* you can
pass around, store, and only later run. `let it = (0..).map(|x| x*x)` is an
infinite iterator that costs nothing until something consumes a finite prefix of
it (`take(5)`). There's no Go expression that is "a computation not yet run"
in that first-class way.*

---

## 6. `for` is sugar for an iterator; ranges

A `for` loop in Rust is *defined* as iterator consumption. `for x in thing`
calls `thing.into_iter()` and loops over `next()` until it yields `None`:

```rust
fn main() {
    let v = vec![10, 20, 30];

    for x in &v {        // &v -> v.iter(),     x: &i32  (borrows; v survives)
        println!("{x}");
    }
    for x in v {         // v  -> v.into_iter(), x: i32   (moves; v consumed)
        println!("{x}");
    }
    // v is gone now.
}
```

`for x in &v` is shorthand for `v.iter()`, `for x in &mut v` for `v.iter_mut()`,
and `for x in v` for `v.into_iter()`. That's why §4's ownership table matters
even when you never type `.iter()` — the bare `for x in v` moves the collection.

**Ranges** are iterators too, which is how you write a counting loop:

```rust
fn main() {
    for i in 0..5 {      // 0..5 is exclusive: 0 1 2 3 4
        print!("{i} ");  // prints: 0 1 2 3 4
    }
    println!();
    for i in 0..=5 {     // 0..=5 is inclusive: 0 1 2 3 4 5
        print!("{i} ");  // prints: 0 1 2 3 4 5
    }
    println!();
}
```

```go
// Go — the C-style three-part loop, or 1.22+ range-over-int
for i := 0; i < 5; i++ { fmt.Print(i, " ") }
for i := range 5 { fmt.Print(i, " ") } // Go 1.22+, also exclusive
```

`0..n` is the everyday "do this n times" / "index from 0 to n-1" loop. It's an
iterator, so it composes with adapters like everything else:
`(0..100).filter(|n| n % 7 == 0).count()` counts multiples of 7 under 100
without ever building a collection.

---

## 7. `collect`: turning a pipeline back into a collection

`collect()` is the consumer that materializes an iterator into a container. It's
generic over the target type, so you tell it (via a type annotation or the
turbofish `::<>`) what to build:

```rust
use std::collections::{HashMap, HashSet};

fn main() {
    // Into a Vec:
    let squares: Vec<i32> = (1..=5).map(|n| n * n).collect();
    println!("{squares:?}"); // prints: [1, 4, 9, 16, 25]

    // Into a String (collecting chars):
    let shout: String = "hi".chars().map(|c| c.to_ascii_uppercase()).collect();
    println!("{shout}"); // prints: HI

    // Into a HashMap (collecting (K, V) pairs):
    let map: HashMap<i32, i32> = (1..=3).map(|n| (n, n * n)).collect();
    println!("{:?}", map.get(&3)); // prints: Some(9)

    // Into a HashSet (dedupes):
    let set: HashSet<i32> = vec![1, 1, 2, 3, 3].into_iter().collect();
    println!("{}", set.len()); // prints: 3

    // The turbofish form, when there's no binding to annotate:
    let n = (1..=10).filter(|x| x % 2 == 0).collect::<Vec<_>>().len();
    println!("{n}"); // prints: 5
}
```

The trick that ties back to Part 6 (`06-error-handling.md`): collecting an
iterator of `Result`s into a `Result<Vec<_>, _>` **short-circuits on the first
error**. It turns "a sequence of fallible operations" into "one fallible
operation over the whole sequence" — no manual loop with an early `return Err`:

```rust
fn main() {
    let inputs = vec!["1", "2", "3"];
    // collect into Result<Vec<i32>, _>: Ok(vec) only if EVERY parse succeeds
    let parsed: Result<Vec<i32>, _> =
        inputs.iter().map(|s| s.parse::<i32>()).collect();
    println!("{parsed:?}"); // prints: Ok([1, 2, 3])

    let bad = vec!["1", "nope", "3"];
    let parsed2: Result<Vec<i32>, _> =
        bad.iter().map(|s| s.parse::<i32>()).collect();
    println!("{}", parsed2.is_err()); // prints: true — stopped at "nope"
}
```

```go
// Go — the hand-written equivalent: loop, parse, bail on first error
func parseAll(inputs []string) ([]int, error) {
    out := make([]int, 0, len(inputs))
    for _, s := range inputs {
        n, err := strconv.Atoi(s)
        if err != nil {
            return nil, err // early return on first failure
        }
        out = append(out, n)
    }
    return out, nil
}
```

`collect::<Result<Vec<_>, _>>()` *is* that whole Go function, as one method
call. This pattern — fallible map then collect into `Result` — is everywhere in
real Rust, and it's the cleanest example of iterators and the error system
composing.

---

## 8. Closures: anonymous functions that capture

A **closure** is an anonymous function written `|args| body`. You've been
feeding them to `map` and `filter` already:

```rust
fn main() {
    let add_one = |x: i32| x + 1;          // explicit param type
    let add = |a, b| a + b;                // types inferred from use
    println!("{}", add_one(4));            // prints: 5
    println!("{}", add(2, 3));             // prints: 5

    // The point of closures: they capture the surrounding environment.
    let factor = 10;
    let scale = |x: i32| x * factor;       // captures `factor` from outside
    println!("{}", scale(5));              // prints: 50
}
```

```go
// Go
func main() {
    addOne := func(x int) int { return x + 1 }
    fmt.Println(addOne(4)) // 5

    factor := 10
    scale := func(x int) int { return x * factor } // captures factor
    fmt.Println(scale(5)) // 50
}
```

So far, identical to Go's function literals. The divergence is **how** the
capture works. Go closures capture variables *by reference* and let the GC keep
them alive as long as the closure does — capture is invisible and always works.
Rust closures capture by the *least* access they need (a `&` borrow if reading,
`&mut` if mutating, a move if they must own it), and that capture is subject to
the borrow checker. This is the source of the famous closure-borrow errors:

```rust
fn main() {
    let mut data = vec![1, 2, 3];

    let mut push = |x| data.push(x); // captures `data` by &mut borrow
    push(4);
    // println!("{:?}", data);       // ERROR: cannot borrow `data` while the
                                     // closure's &mut borrow is still live
    push(5);
    println!("{data:?}");            // ok now: closure's borrow has ended
                                     // prints: [1, 2, 3, 4, 5]
}
```

⚡ *Where the Go analogy breaks: in Go you never think about what a closure
captures or how — the GC makes it a non-issue, and a closure can freely outlive
the function that created it. In Rust the capture mode is part of the program's
borrow structure: a closure that mutably borrows `data` is, for its lifetime, an
exclusive borrow of `data`, and nothing else may touch `data` until the closure
is done. Most "weird closure errors" beginners hit are this rule, not a closure
bug.*

### `move`: forcing capture by value

When a closure needs to *own* what it captures — most importantly because it
will outlive the current scope, like being sent to a thread — prefix it with
`move`:

```rust
fn main() {
    let name = String::from("ada");
    let greet = move || println!("hi {name}"); // `name` is MOVED into the closure
    greet();                                    // prints: hi ada
    // println!("{name}");                      // ERROR: name was moved into greet
}
```

`move` is what you'll reach for constantly in Part 9
(`09-concurrency.md`): a closure handed to `thread::spawn` must own its
captures, because the thread may run after the spawning function returns, so
borrowing from that stack frame would dangle. `move` makes the closure
self-contained. Go's `go func() { ... }()` captures by reference and trusts the
GC; Rust's `move` makes ownership transfer explicit precisely because there's no
GC to clean up after.

---

## 9. `Fn`, `FnMut`, `FnOnce`: the three closure traits

Every closure automatically implements one or more of three traits, and which
ones depend on *how it uses* its captures. This is how you write functions that
accept or store closures.

```rust
// Fn:     borrows captures with &        — callable many times, read-only
// FnMut:  borrows captures with &mut     — callable many times, mutates state
// FnOnce: takes captures by value (move) — callable AT MOST ONCE (consumes them)
```

Every closure is at least `FnOnce`. It's also `FnMut` if it doesn't move out its
captures, and also `Fn` if it doesn't even mutate them. So `Fn` ⊂ `FnMut` ⊂
`FnOnce` in capability ordering, and you bound on the *loosest* trait your
function actually needs.

```rust
// Accept a closure by generic bound (preferred — monomorphizes, zero-cost):
fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

// Same thing with impl Trait syntax — cleaner for simple signatures:
fn apply2(f: impl Fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

// A closure that mutates its environment needs FnMut:
fn run_twice<F: FnMut()>(mut f: F) {
    f();
    f();
}

fn main() {
    println!("{}", apply(|x| x + 1, 10));  // prints: 11
    println!("{}", apply2(|x| x * 2, 10)); // prints: 20

    let mut count = 0;
    run_twice(|| count += 1); // FnMut: mutates `count`
    println!("{count}");      // prints: 2
}
```

```go
// Go — one func type covers all of this; capture mode is never expressed
func apply(f func(int) int, x int) int { return f(x) }

func main() {
    fmt.Println(apply(func(x int) int { return x + 1 }, 10)) // 11
    count := 0
    twice := func() { count++ }
    twice(); twice()
    fmt.Println(count) // 2
}
```

Go has exactly one notion: a `func` value. It captures by reference, can be
called any number of times, and the type signature says nothing about what it
touches. Rust splits that into three traits because the *ownership* of the
captures differs: a closure that moves a `String` out of its environment can
only run once (the second call would use a moved value), so it's `FnOnce` and
the type system stops you calling it twice.

To **store** a closure (in a struct field, a `Vec`, a return type) where the
concrete type is unknown, box it as a trait object — `Box<dyn Fn(...)>`:

```rust
struct Button {
    on_click: Box<dyn Fn()>, // a stored closure of unknown concrete type
}

fn main() {
    let label = String::from("save");
    let b = Button {
        on_click: Box::new(move || println!("clicked {label}")),
    };
    (b.on_click)(); // prints: clicked save
}
```

The generic-bound form (`impl Fn`, `<F: Fn>`) is zero-cost and monomorphizes
(Part 5); the `Box<dyn Fn>` form is a heap-allocated trait object with dynamic
dispatch — use it when you need to store closures of differing types in one
place (e.g. a `Vec<Box<dyn Fn()>>` of callbacks). It's the same
static-vs-dynamic dispatch trade-off as `impl Trait` vs `Box<dyn Trait>` from
Part 5 (`05-traits-and-generics.md`).

---

## 10. The patterns you'll actually type

A grab-bag of the iterator idioms that show up constantly, each replacing a Go
loop you'd otherwise hand-write.

```rust
fn main() {
    let words = vec!["apple", "12", "fig", "7", "kiwi"];

    // filter_map: filter + map in one — keep items where the closure is Some.
    // Here: keep the strings that parse as numbers, as their parsed value.
    let nums: Vec<i32> = words.iter().filter_map(|s| s.parse().ok()).collect();
    println!("{nums:?}"); // prints: [12, 7]

    // fold: a running accumulator (Go's "loop with an acc variable").
    let total = (1..=5).fold(0, |acc, n| acc + n);
    println!("{total}"); // prints: 15  (this is also just .sum())

    // enumerate: pairs of (index, item) — Go's `for i, x := range`.
    for (i, w) in words.iter().enumerate() {
        if i == 0 { println!("first is {w}"); } // prints: first is apple
    }

    // zip: walk two iterators in lockstep, stopping at the shorter.
    let names = ["ada", "bob"];
    let ages = [36, 41];
    let pairs: Vec<_> = names.iter().zip(ages.iter()).collect();
    println!("{pairs:?}"); // prints: [("ada", 36), ("bob", 41)]

    // position / find / any / all — search consumers.
    let idx = words.iter().position(|w| *w == "fig"); // Some(2)
    let big = nums.iter().any(|n| *n > 10);           // true
    println!("{idx:?} {big}"); // prints: Some(2) true
}
```

Sorting is method-based and in place. `sort_by_key` is the one you'll use most —
the same shape as `sort.Slice` in Go but with a key function instead of a less
function:

```rust
fn main() {
    let mut people = vec![("ada", 36), ("bob", 41), ("cy", 22)];
    people.sort_by_key(|&(_, age)| age); // sort ascending by age
    println!("{people:?}"); // prints: [("cy", 22), ("ada", 36), ("bob", 41)]

    let mut nums = vec![3, 1, 2];
    nums.sort();                  // sort() for the natural ordering
    nums.sort_by(|a, b| b.cmp(a)); // sort_by for a custom comparator (descending)
    println!("{nums:?}");          // prints: [3, 2, 1]
}
```

```go
// Go
sort.Slice(people, func(i, j int) bool { return people[i].age < people[j].age })
sort.Ints(nums)
```

Finally, slice **windowing** — `windows(n)` (overlapping) and `chunks(n)`
(non-overlapping) — which Go has no built-in for at all:

```rust
fn main() {
    let v = [1, 2, 3, 4, 5];
    for w in v.windows(2) {
        println!("{w:?}"); // prints: [1, 2] then [2, 3] then [3, 4] then [4, 5]
    }
    for c in v.chunks(2) {
        println!("{c:?}"); // prints: [1, 2] then [3, 4] then [5]
    }
}
```

`windows` is perfect for "compare each element to its neighbor"; `chunks` for
batching. In Go you'd index with `v[i]` and `v[i+1]` and babysit the bounds.

---

## 11. Zero-cost: the chain *is* the loop

The objection a performance-minded Go developer raises at this point: "all those
`map`/`filter`/`collect` calls must allocate intermediate slices and add
overhead, right?" No. Iterator adapters are **zero-cost abstractions**. Each
adapter is a tiny struct with an inlined `next`, and when you bound a generic on
`Iterator` the compiler **monomorphizes** (Part 5) the whole chain into one
concrete type, then inlines and optimizes it into a single tight loop — no
intermediate `Vec`, no closure-call overhead, no heap allocation for the
pipeline itself.

```rust
// This chain...
let sum: u64 = (1..=1_000_000).filter(|n| n % 2 == 0).map(|n| n as u64).sum();

// ...compiles to essentially the same machine code as this hand loop:
let mut sum: u64 = 0;
for n in 1..=1_000_000u64 {
    if n % 2 == 0 { sum += n; }
}
```

There's no laziness penalty either — laziness *helps* here, because no
intermediate collection is ever built; items flow one at a time through the
whole chain. So the readable, declarative version is also the fast version. You
don't trade performance for expressiveness, which is the recurring Rust promise
and the reason reaching for an iterator chain over a hand loop is a free win.

⚡ *Where the Go analogy breaks: even if Go's stdlib had `Map`/`Filter`, they'd
likely return new slices (allocating) or rely on dynamic dispatch through
`func` values that the compiler can't always inline — a real cost, which is part
of why idiomatic Go stays with explicit loops. Rust's monomorphization makes the
abstraction genuinely free, so the language can build its whole iteration
culture on it without a performance guilty conscience.*

---

## One-sentence mental model

Collections are owned containers whose `iter` / `iter_mut` / `into_iter` decide
whether you borrow, mutate, or consume their elements; iterator **adapters**
(`map`, `filter`, ...) lazily describe a pipeline that does nothing until a
**consumer** (`collect`, `sum`, `for`, ...) drives it, monomorphizing into a
loop as tight as anything you'd hand-write in Go — and closures are the
capturing functions you feed that pipeline, with `Fn`/`FnMut`/`FnOnce` encoding,
in the type system, how they touch the environment Go would just hand to the GC.

---

[← Error handling](06-error-handling.md) · [Next: Smart pointers and interior mutability →](08-smart-pointers-and-interior-mutability.md)
