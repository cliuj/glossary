# Part 2 — Ownership and borrowing

[← Reading Rust](01-reading-rust.md) · [Next: Lifetimes →](03-lifetimes.md)

This is the part of the guide everything else leans on. Ownership and borrowing
are the single idea that makes Rust *Rust* — the source of most of what
surprises a Go developer, and the foundation under lifetimes (Part 3), errors
(Part 6), iterators (Part 7), smart pointers (Part 8), and concurrency (Part
9). Read it slowly. If a later chapter confuses you, the confusion is almost
always an unpaid debt from this one.

---

## 1. Why ownership exists at all

In Go you almost never think about who frees memory. You allocate, you share
pointers around, you return slices from functions, and the garbage collector
reclaims everything once nothing references it. The cost is a runtime that
periodically walks your heap, plus the small, occasional latency of a GC pause.
The benefit is that you essentially never get aliasing or freeing wrong — the
GC makes "who owns this?" a non-question.

Rust deletes the garbage collector. There is no runtime walking the heap, no
pause, nothing tracking liveness at runtime. So the question Go's GC quietly
answers — *when is it safe to free this, and who does it?* — has to be answered
some other way. Rust's answer is **ownership**: a set of rules the compiler
enforces statically, so that the moment a value's owner goes out of scope, the
compiler can insert exactly one `free`, and prove no live reference outlives
it.

```rust
fn main() {
    let s = String::from("hello"); // s owns a heap allocation
    println!("{s}");
} // s goes out of scope here; its heap buffer is freed right here, deterministically
```

```go
// Go
func main() {
    s := "hello"      // backing bytes live on the heap if they escape
    fmt.Println(s)
}                     // nothing freed here; the GC reclaims it sometime later, off to the side
```

The Go version frees *eventually*; the Rust version frees *at the closing
brace*, every time, with no background machinery. That determinism — memory,
file handles, locks, sockets all released at a known point — is the whole
prize, and ownership is the price of admission.

---

## 2. The three rules

All of ownership compresses into three rules the compiler checks:

1. **Each value has exactly one owner.** A `String`, a `Vec<T>`, a struct you
   defined — each is owned by exactly one variable.
2. **There is only one owner at a time.** Ownership can be *transferred*
   (moved), but it is never shared. After a move the previous owner is gone.
3. **When the owner goes out of scope, the value is dropped.** "Dropped" means
   its destructor runs and its resources are released.

That third rule is RAII — *resource acquisition is initialization* — the same
pattern behind C++ destructors and Go's `defer`, except it's automatic and tied
to scope rather than something you remember to write. A type can customize what
"dropped" means by implementing the `Drop` trait (full treatment in Part 8);
for now, know that dropping a `String` frees its buffer, dropping a `File`
closes the handle, dropping a `MutexGuard` releases the lock — all at the
closing brace, in reverse order of creation.

```rust
{
    let a = String::from("first");
    let b = String::from("second");
    // ... use a and b ...
} // b dropped first, then a — reverse declaration order
```

```go
// Go
func f() {
    a := openFile("first")
    defer a.Close()        // you opt in, explicitly, per resource
    b := openFile("second")
    defer b.Close()        // defers run LIFO, like Rust's drop order
}
```

⚡ *Where the Go analogy breaks: Go's `defer` is opt-in cleanup you attach by
hand and can forget; Rust's drop is not a feature you invoke — it's structural.
Every owned value is dropped, always, exactly once, whether or not you thought
about it. You can't "forget to `defer`" because there's nothing to remember.*

---

## 3. Move semantics

Here is the line that trips up every Go developer on day one:

```rust
let s1 = String::from("hello");
let s2 = s1;          // s1 is MOVED into s2
println!("{s1}");     // ERROR: borrow of moved value: `s1`
```

In Go, `s2 := s1` copies the string header (a pointer, a length) and both
variables remain perfectly usable — they point at the same backing bytes, and
the GC keeps those bytes alive as long as either is reachable.

```go
// Go
s1 := "hello"
s2 := s1            // copies the header; both s1 and s2 are fine
fmt.Println(s1)     // totally legal
fmt.Println(s2)
```

In Rust, `let s2 = s1;` *moves* ownership. A `String` is a three-word header on
the stack — pointer, length, capacity — pointing at a heap buffer. After the
move, `s2` holds that header and `s1` is statically marked invalid. Rust does
*not* copy the buffer, and it does *not* let both names live, because then rule
2 (one owner) would break and the compiler couldn't know which variable's
scope-exit should free the buffer. So it picks one: the new one. The old name
is dead.

This is the same shallow copy Go does — pointer, length, capacity — but Rust
adds one thing on top: it invalidates the source. That single extra rule is
what makes a double-free impossible.

> **Trap:** "Move" sounds expensive, like it's copying data around. It isn't. A
> move is a memcpy of the *header* (a few words) plus a compile-time note that
> the old name is dead. It's exactly as cheap as Go's assignment — the only
> difference is you can't touch the source afterward. Moving a `Vec` of a
> million elements moves three words, not a million.

Passing to a function moves too:

```rust
fn consume(s: String) {
    println!("{s}");
} // s dropped here

fn main() {
    let name = String::from("Ada");
    consume(name);            // name MOVED into the function
    // println!("{name}");    // ERROR: borrow of moved value: `name`
}
```

The function took *ownership*. When `consume` ends, it drops the string. The
caller no longer has it. In Go you'd pass the string and keep using it without
a thought — the move discipline simply doesn't exist there.

---

## 4. `Copy` types: the values that don't move

Not everything moves. Integers do this, and it's fine:

```rust
let x = 5;
let y = x;        // x is COPIED, not moved
println!("{x} {y}"); // prints: 5 5  — both usable
```

Why does this work when `String` didn't? Because `i32` implements the `Copy`
trait. A `Copy` type is one that's entirely a fixed-size blob of bytes on the
stack with no heap buffer, no owned resource, nothing that a destructor needs
to clean up. Duplicating its bytes produces a second fully-valid,
fully-independent value — so there's no ownership to track and no reason to
invalidate the source. Assigning a `Copy` value just copies the bits, exactly
like Go assigning an `int`.

What's `Copy`:

- All the integer types (`i32`, `u64`, `usize`, …), `f32`/`f64`, `bool`,
  `char`.
- Shared references `&T` (a `&T` is just a pointer; copying the pointer is
  fine).
- Tuples and fixed-size arrays *whose elements are all `Copy`*: `(i32, bool)`
  is `Copy`; `(i32, String)` is not.

What's *not* `Copy`: `String`, `Vec<T>`, `HashMap`, `Box<T>`, and any struct
you define (unless you derive `Copy`, which the compiler only allows if every
field is `Copy`). These own a heap resource, so bit-copying them would create
two owners of one buffer — exactly the double-free ownership is built to
forbid. So they move instead.

```go
// Go
x := 5
y := x          // copies the int; both usable
// In Go EVERY assignment is a copy of the value's representation.
// For an int that's the int; for a slice/map/string it's the header.
// Both copies stay valid because the GC, not scope, decides freeing.
```

⚡ *Where the Go analogy breaks: in Go the value/reference distinction is about
the type (slices and maps are reference-y, ints are value-y), but **every**
assignment leaves the source usable. In Rust the distinction is `Copy` vs.
move, and only `Copy` leaves the source usable. A Go slice header and a Rust
`Vec` header are nearly the same three words — the difference is purely that
Rust kills the source name on assignment and Go doesn't.*

---

## 5. `Clone`: deep copies, on demand

When you genuinely want a second independent `String` or `Vec` — buffer and all
— you ask for it explicitly with `.clone()`:

```rust
let s1 = String::from("hello");
let s2 = s1.clone();     // allocates a NEW buffer, copies the bytes
println!("{s1} {s2}");   // prints: hello hello  — both valid, independent
```

`Clone` is the deep, possibly-expensive copy; `Copy` is the cheap, implicit,
bitwise one. The visibility is the point: a `.clone()` on a big `Vec` is a real
allocation and a real memcpy, and Rust makes you write it so the cost is never
hidden. Go has no such marker — copying a slice's contents is `append([]T(nil),
src...)` or `copy(...)`, and a plain `s2 := s1` deceptively shares the backing
array, a classic Go aliasing footgun.

> **Trap:** `.clone()` is the beginner's escape hatch, and the whole community
> knows it. When the borrow checker rejects your code, sprinkling `.clone()`
> until it compiles *works* — and quietly reintroduces the copying overhead you
> came to Rust to avoid. It's fine while learning. But if your instinct for
> every borrow error is "clone it," you're writing Go-with-extra-steps. The
> rest of this chapter is about the better answer: borrowing.

---

## 6. Borrowing: references instead of giving it away

The "give it back" pattern shows why borrowing exists. Without references, a
function that needs to *look* at your data has to take ownership and hand it
back:

```rust
fn len_giving_back(s: String) -> (String, usize) {
    let n = s.len();
    (s, n)            // return the string so the caller can keep using it
}

fn main() {
    let s = String::from("hello");
    let (s, n) = len_giving_back(s);   // tedious shuffle
    println!("{s} has length {n}");
}
```

Nobody writes that. Instead you **borrow** — lend the function a reference
without transferring ownership:

```rust
fn len(s: &String) -> usize {  // borrows; does not own
    s.len()
}

fn main() {
    let s = String::from("hello");
    let n = len(&s);            // lend a reference
    println!("{s} has length {n}"); // s still valid — never moved
}
```

`&s` creates a **shared reference**: a pointer that grants read access but not
ownership. When `len` returns, nothing is dropped — `len` never owned anything.
The owner, `main`, keeps `s`.

If a Go developer squints, `&s` looks like `&s` in Go — and the syntax is
identical. The difference is what the compiler guarantees *about* that pointer,
which is §8.

### `&mut`: borrowing to mutate

A shared reference is read-only. To change borrowed data you need a **mutable
reference**, `&mut T`, and the owner must itself be `mut`:

```rust
fn push_world(s: &mut String) {
    s.push_str(" world");   // auto-deref: (*s).push_str(...)
}

fn main() {
    let mut s = String::from("hello");
    push_world(&mut s);
    println!("{s}"); // prints: hello world
}
```

To read through a reference you sometimes dereference explicitly with `*`:

```rust
let x = 10;
let r = &x;
println!("{}", *r);        // prints: 10
let mut n = 1;
let m = &mut n;
*m += 41;                  // write through the reference
println!("{n}");           // prints: 42
```

Method calls and `{}` formatting auto-deref, so you write `s.len()` not
`(*s).len()`; you mostly reach for `*` when reading or writing a referenced
scalar.

```go
// Go
func pushWorld(s *string) {
    *s = *s + " world"     // explicit deref to assign
}
func main() {
    s := "hello"
    pushWorld(&s)
    fmt.Println(s)         // hello world
}
```

Go has `&` and `*` and pointers, so the *mechanics* feel familiar. What Go does
not have is any rule about how many of those pointers may exist at once, or
whether they may mutate. That rule is the entire game.

---

## 7. The borrow rules

At any given moment, for any given value, you may have **either**:

- **any number of shared references `&T`** (read-only, freely aliased), **XOR**
- **exactly one mutable reference `&mut T`** (read-write, exclusive),

and **never both at the same time**. On top of that: **every reference must
always point at valid data** — no reference may outlive the value it points to
(no dangling pointers, ever).

That's it. "Many readers or one writer, never both." If you've used a
`sync.RWMutex` in Go, the shape is familiar — except a `RWMutex` enforces it at
*runtime* with locks, and Rust enforces it at *compile time* with no runtime
cost whatsoever.

Why those exact rules? Because "one writer, no concurrent readers" is precisely
the condition that makes a **data race impossible**. A data race needs two
accesses to the same location, at least one a write, unsynchronized. The borrow
rules make that unrepresentable: if someone holds `&mut`, no one else holds any
reference; if anyone holds `&`, no one holds `&mut`. The compiler proves the
absence of data races structurally — and the same rule that prevents
*concurrent* aliasing bugs also prevents *single-threaded*
iterator-invalidation bugs (§9), which is why it applies even in code with no
threads at all.

```go
// Go
// Go has NO compile-time equivalent. You guard shared mutable state with a
// mutex by convention, and catch the races you forgot at RUNTIME:
//     go test -race ./...
// The -race detector is excellent — but it only flags races on code paths
// that actually executed during the run. Rust rejects them before main() runs.
```

⚡ *Where the Go analogy breaks: in Go, `go test -race` is a probabilistic,
runtime tool — it finds the races your test happened to trigger. Rust's borrow
checker is a total, compile-time proof — it rejects the *category*. The flip
side: Rust will also reject perfectly safe programs it can't prove safe, and
you'll occasionally restructure code (or reach for Part 8's `RefCell`) to
convince it. Go never blocks you at compile time for aliasing; Rust sometimes
blocks you for aliasing that would actually have been fine.*

### Non-lexical lifetimes: a borrow ends at its last use

The rules sound strict, but they're more forgiving than they read, because a
borrow doesn't live until the end of its scope — it lives until its **last
use**. This is called NLL (non-lexical lifetimes), and it's why a lot of
"surely this is illegal" code just compiles:

```rust
let mut s = String::from("hello");

let r = &s;            // shared borrow starts
println!("{r}");       // ... and ends HERE, at its last use

let m = &mut s;        // mutable borrow — fine, r is already done
m.push_str(" world");
println!("{m}");       // prints: hello world
```

Lexically, `r` and `m` coexist in the same block, which looks like a violation.
But the compiler sees that `r` is never touched after the `println!`, so its
borrow has already ended by the time `m` is created. No overlap, no error.
You'll lean on this constantly without thinking about it — just know that "the
borrow ends where you stop using the reference," not at the `}`.

---

## 8. The classic aliasing error

Here's the canonical case where the borrow checker stops you, and it's worth
internalizing because it's not arbitrary:

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];     // shared borrow of an element
v.push(4);             // ERROR: cannot borrow `v` as mutable because
                       //        it is also borrowed as immutable
println!("{first}");   // first is used here, so the borrow is still live
```

`first` is a `&` into `v`'s buffer. `v.push(4)` needs `&mut v`, because pushing
might force the `Vec` to **reallocate** — grab a bigger buffer, copy the
elements, free the old one. If that happened, `first` would point into the
freed old buffer: a dangling pointer, a use-after-free. The borrow checker
forbids it at compile time.

```go
// Go
s := []int{1, 2, 3}
first := &s[0]
s = append(s, 4)     // may reallocate the backing array
fmt.Println(*first)  // compiles fine — but `first` may now point at the OLD
                     // array. The GC keeps that old array alive, so it's not a
                     // crash, but `first` no longer reflects `s`. Silent bug.
```

This is the heart of it: the *exact same operation* is a silent correctness bug
in Go (your pointer quietly aliases a stale backing array the GC keeps alive)
and a compile error in Rust. Rust would rather refuse to build than let `first`
go stale.

> **Trap:** you'll hit this whenever you hold a reference into a collection and
> then try to modify the collection — `push`, `insert`, `remove`, even sorting.
> The fix is usually to finish with the reference first (NLL helps), copy out
> the value you need (`let first = v[0];` since `i32` is `Copy`), or
> restructure so the reads and the write don't overlap. Reaching for `.clone()`
> here is the lazy fix; restructuring is the Rust-fluent one.

> **Trap:** dangling references are caught even without collections. Returning
> a reference to a local is the textbook case:
> ```rust
> fn dangle() -> &String {
>     let s = String::from("oops");
>     &s            // ERROR: missing lifetime specifier / returns a reference
> }                 //        to data owned by the current function
> ```
> `s` is dropped when `dangle` returns, so the reference would point at freed
> memory. Rust rejects it. Return the `String` itself (move it out) instead of
> a reference to it. Part 3 explains the lifetime machinery behind this error.

---

## 9. Slices: borrowed views into owned data

A **slice** is a borrowed window into a contiguous run of someone else's owned
data. It's a reference — pointer plus length — that owns nothing:

- `&str` is a slice into a `String` (or a string literal's static memory).
- `&[T]` is a slice into a `Vec<T>` or an array.

```rust
let s = String::from("hello world");
let hello: &str = &s[0..5];   // borrows bytes 0..5 of s — owns nothing
let world: &str = &s[6..11];
println!("{hello} {world}");  // prints: hello world

let v = vec![10, 20, 30, 40];
let middle: &[i32] = &v[1..3]; // a view, not a copy
println!("{middle:?}");        // prints: [20, 30]
```

Because a slice is a borrow, the borrow rules apply to it fully. A slice
**cannot outlive the data it points into**, and you can't mutate the owner
while a slice of it is live:

```rust
let mut s = String::from("hello world");
let word = &s[0..5];   // borrows s
s.clear();             // ERROR: cannot borrow `s` as mutable...
                       //        `word` is still borrowing it
println!("{word}");
```

`s.clear()` empties the string; `word` pointed into those bytes. Same shape as
the `Vec::push` case in §8 — a live borrow blocks mutation that could
invalidate it.

```go
// Go
s := []int{10, 20, 30, 40}
middle := s[1:3]       // a Go slice: ptr + len + cap, sharing s's backing array
s = append(s, 50)      // may or may not reallocate; if it doesn't, this MUTATES
                       // what `middle` sees. Aliasing is silent and on you.
fmt.Println(middle)
```

⚡ *Where the Go analogy breaks: a Go slice header carries `cap` and an implicit
claim on the backing array — the GC keeps that array alive as long as any slice
references it, so a Go slice can freely outlive the variable it came from. A
Rust slice is a pure borrow with no claim on anything: it cannot outlive its
owner, the compiler enforces that, and there's no GC underneath to paper over a
stale alias. Go slices keep data alive; Rust slices are kept alive by data.*

The `&str` / `String` split (and `&[T]` / `Vec<T>`) is the borrowed-vs-owned
distinction made concrete, and it's why idiomatic Rust functions take `&str`
and `&[T]` parameters: they accept a borrow of *anything* — a `String`, a
literal, a sub-slice — without forcing the caller to give up ownership.

---

## 10. When single ownership isn't enough — a forward pointer

The whole chapter assumes one owner per value. Sometimes you genuinely need
shared ownership — a value with no single clear owner, kept alive until the
last user is done. The borrow checker won't let you fake that with references.
The real tools, covered in Part 8, are:

- **`Rc<T>`** — reference counting for single-threaded shared ownership. The
  closest Rust gets to "the GC keeps it alive while anyone holds it," except
  the count is explicit and freeing is still deterministic (at the last
  `drop`).
- **`Arc<T>`** — the same, atomic, for sharing across threads (Part 9).
- **`RefCell<T>`** — moves the borrow check from compile time to *runtime*, so
  you can mutate through a shared reference; it `panic!`s if you break the
  borrow rules. The escape hatch when the static checker is too conservative
  for a genuinely-valid pattern.

And **lifetimes** (Part 3) are the notation for the rule you met in §8's second
trap: they let you *name* how long a borrow is valid so the compiler can check
references that cross function boundaries. Everything in this chapter was the
borrow checker working with lifetimes it could infer; Part 3 is what happens
when you have to spell them out.

---

## One-sentence mental model

Every value has exactly one owner and is freed deterministically when that
owner's scope ends; you may *move* ownership elsewhere (killing the old name)
or *borrow* it out as many shared `&T` readers **xor** one exclusive `&mut T`
writer — and the compiler proves, at build time and with zero runtime cost,
that no borrow ever dangles and no two writers ever alias, which is the same
guarantee Go buys at runtime with the GC and the `-race` detector.

---

[← Reading Rust](01-reading-rust.md) · [Next: Lifetimes →](03-lifetimes.md)
