# Part 8 — Smart pointers and interior mutability

[← Collections, iterators, closures](07-collections-iterators-closures.md) · [Next: Concurrency →](09-concurrency.md)

Single ownership (Part 2) and the borrow rules are restrictive **on purpose**:
one owner, and either many readers or one writer. That covers most code, but
some shapes — recursive types, graphs with shared nodes, mutating data several
owners can see — don't fit. This part is the catalogue of deliberate, explicit
escape valves for exactly those shapes. In Go you never reach for any of this,
because the GC plus freely-shared references handle it invisibly; in Rust you
pick the exact sharing strategy and pay only for the one you chose.

---

## 1. The shape of the problem

Everything in this part exists to relax one of two constraints: **where a value
lives / who owns it** (`Box`, `Rc`, `Arc`) or **when its borrows are checked**
(`Cell`, `RefCell`, `Mutex`). Go collapses both questions into "it's a pointer,
the GC sorts it out." Rust makes you name the trade-off:

| You want…                              | Reach for           |
| -------------------------------------- | ------------------- |
| Heap allocation, one owner             | `Box<T>`            |
| Many owners, single-threaded           | `Rc<T>`             |
| Many owners, across threads            | `Arc<T>`            |
| Mutate through a shared, immutable ref | `RefCell<T>` / `Cell<T>` |
| …the same, across threads              | `Mutex<T>` / `RwLock<T>` |

The combinations matter more than the pieces: `Rc<RefCell<T>>` and
`Arc<Mutex<T>>` are the two you'll actually type. We build up to them.

---

## 2. `Box<T>`: one owner, on the heap

`Box<T>` is the simplest smart pointer: a heap allocation with a single owner.
The `Box` *is* the owner; when it drops, the heap value is freed. It's a thin
pointer with no runtime cost beyond the allocation itself.

```rust
let b = Box::new(5);   // 5 lives on the heap; b owns it
println!("{}", *b);    // prints: 5 — deref to read the value
// b drops here, heap freed
```

There are three reasons you actually reach for `Box`.

**Recursive types.** Rust must know every type's size at compile time. A
straight recursive `enum` is infinitely sized — the compiler can't lay it out:

```rust
enum List {
    Cons(i32, List), // ERROR: recursive type `List` has infinite size
    Nil,
}
```

`Box` fixes it: a `Box<List>` is just a pointer, which has a known, fixed size,
no matter how big the thing it points to is.

```rust
enum List {
    Cons(i32, Box<List>), // a pointer-sized field — size is now known
    Nil,
}

use List::{Cons, Nil};
let list = Cons(1, Box::new(Cons(2, Box::new(Nil))));
```

```go
// Go
type List struct {
    Val  int
    Next *List // a pointer is fine; Go never asks "what's the inline size?"
}

list := &List{1, &List{2, nil}}
```

⚡ *Where the Go analogy breaks: in Go a struct field that's a pointer to its own
type "just works," because a pointer is always one word and Go never tries to
inline the pointee. Rust would happily inline the value if you let it — that's
the default — so you have to opt into indirection with `Box`. The need for
`Box` here isn't about the heap; it's about making the size finite.*

**Large values.** Moving a big value means a `memcpy`. `Box` it and you move a
pointer instead. Rarely the first thing you reach for, but real.

**Trait objects.** This is the big one and ties back to traits (Part 5,
`05-traits-and-generics.md`). A `Box<dyn Trait>` is an owned, heap-allocated
value of *some* type implementing `Trait`, with the concrete type erased behind
a vtable. It's how you get a heterogeneous collection — the Rust equivalent of
a Go slice of an interface type:

```rust
trait Draw { fn draw(&self); }

let shapes: Vec<Box<dyn Draw>> = vec![
    Box::new(Circle { r: 1.0 }),
    Box::new(Square { side: 2.0 }),
];
for s in &shapes { s.draw(); } // dynamic dispatch through the vtable
```

```go
// Go
type Draw interface{ Draw() }

shapes := []Draw{ Circle{1.0}, Square{2.0} } // an interface IS a fat pointer
for _, s := range shapes { s.Draw() }
```

A Go interface value is already a (type, data) fat pointer that boxes its
contents on the heap when needed; `Box<dyn Draw>` is the explicit spelling of
that same machinery. The difference: in Go *everything* is implicitly boxable
and escape analysis decides heap-vs-stack for you. In Rust the default is
stack/inline, and `Box` is you saying "heap this, and erase the type."

---

## 3. `Deref`: why smart pointers feel transparent

You wrote `*b` above and called `s.draw()` on a `Box`. Both work because `Box`
implements the `Deref` trait (and `DerefMut` for the mutable version). `Deref`
is what lets a custom type stand in for a reference to the thing inside it.

Two ergonomics fall out of it. First, the `*` operator: `*b` on a
`Box<i32>` gives you the `i32`. Second, and more important day to day, **deref
coercion**: when you call a method or pass an argument, the compiler will
insert as many `*` as needed to make the types line up.

```rust
fn greet(name: &str) { println!("Hi, {name}"); }

let s: String = String::from("Ada");
greet(&s);    // &String coerces to &str — String: Deref<Target = str>
```

This is why `&String` works where `&str` is expected, why `&Vec<T>` works as
`&[T]`, and why you can call `i32` methods straight through a `Box<i32>`. It's
the mechanism behind a lot of Rust feeling less fiddly than the type signatures
suggest. You'll rarely implement `Deref` yourself outside of writing your own
smart-pointer type — just know it's the reason these wrappers are invisible at
the call site.

> **Trap:** Don't implement `Deref` to fake inheritance by "deref-ing" a struct
> to one of its fields. It compiles, and the auto-deref makes it look like
> subclassing, but it confuses readers and tooling. `Deref` is for
> pointer-like types only.

---

## 4. `Drop`: RAII, gone deeper

Part 2 introduced RAII — a value's destructor runs when its owner goes out of
scope. `Drop` is the trait that lets you customize that cleanup.

```rust
struct Guard(&'static str);

impl Drop for Guard {
    fn drop(&mut self) {
        println!("dropping {}", self.0);
    }
}

fn main() {
    let _a = Guard("a");
    let _b = Guard("b");
    // prints: dropping b
    //         dropping a   — reverse declaration order (LIFO)
}
```

Drop order is **reverse of declaration**, like a stack — last created, first
destroyed. This is deterministic and known at compile time, which is the whole
point.

```go
// Go
func main() {
    defer fmt.Println("dropping a")
    defer fmt.Println("dropping b")
    // prints: dropping b
    //         dropping a   — defers also run LIFO
}
```

The closest Go analogy is `defer`, and the LIFO ordering even matches. But
`defer` is something *you* write at each use site; `Drop` is attached to the
*type* once and fires everywhere that type is used, no caller cooperation
needed. That's why Rust files, locks, and DB handles clean themselves up
without a `defer` at every call site.

You can drop a value early with `std::mem::drop` — handy for releasing a lock
before the end of a scope:

```rust
let g = Guard("early");
drop(g);              // prints: dropping early — right here
println!("after");    // prints: after
// g is gone; no second drop at end of scope
```

> **Trap:** You cannot call `g.drop()` yourself — the method exists but the
> compiler forbids calling it directly, because it would run a *second* time at
> scope end and double-free. Use the free function `std::mem::drop(g)`, which
> takes the value by move so the compiler knows it's spent.

⚡ *Where the Go analogy breaks: Go's runtime finalizers (`runtime.SetFinalizer`)
look like destructors but run at the GC's whim — maybe never, maybe long after
the object is unreachable. `Drop` is synchronous and deterministic: it runs at a
known point in your control flow, not whenever a background collector gets
around to it.*

You rarely implement `Drop` by hand. The standard library already does it for
`Box`, `Vec`, `File`, `MutexGuard`, and so on, and those compose — dropping a
`Vec<File>` drops every `File`. You write `Drop` only when you own a raw
resource the type system doesn't already track.

---

## 5. `Rc<T>`: many owners, single thread

`Box` has exactly one owner. Sometimes a value genuinely needs several — a node
in a graph reachable from multiple places, or shared config a handful of
structs all hold. `Rc<T>` (Reference Counted) gives **shared ownership**: it
keeps a count of how many owners exist, and frees the value when the count hits
zero.

```rust
use std::rc::Rc;

let a = Rc::new("shared".to_string());
let b = Rc::clone(&a);          // count is now 2 — NOT a deep copy
let c = Rc::clone(&a);          // count is now 3
println!("{}", Rc::strong_count(&a)); // prints: 3
// b drops -> 2, c drops -> 2... value freed when the last Rc drops
```

`Rc::clone` is cheap: it bumps an integer and hands back another pointer to the
*same* allocation. It is deliberately written as `Rc::clone(&a)` rather than
`a.clone()` so that a reader sees "this is a refcount bump," not "this deep-copies
a string."

```go
// Go
a := newString("shared")
b := a // just another pointer to the same value; GC tracks reachability
c := a
_ = b; _ = c
// freed sometime after the last reference is gone — when, you don't know
```

In Go, every shared reference is this, automatically and invisibly. `Rc` is you
opting into the same reachability-counting that Go's GC does for everything —
except it's explicit, and the freeing is immediate and deterministic (at count
zero), not eventual.

The catch: a value behind `Rc` is **immutable**. `Rc<T>` only ever hands you
`&T`, never `&mut T` — it can't, because it has no idea how many other owners
are reading right now, and handing out a `&mut` would break the no-aliasing
rule. So how do you mutate shared data? That's the next section.

---

## 6. Interior mutability: `Cell` and `RefCell`

The borrow rules are normally checked at **compile time**. Interior mutability
is the controlled escape hatch: a type that lets you mutate through a shared
`&` reference, moving the borrow check to **run time** instead of giving it up.

`Cell<T>` is the simple case, for `Copy` values: you `get` a copy out and `set`
a new one in. No references hand out, so nothing can be aliased wrongly.

```rust
use std::cell::Cell;

let counter = Cell::new(0);
let r = &counter;        // a shared, immutable reference...
r.set(r.get() + 1);      // ...yet we mutate through it
println!("{}", counter.get()); // prints: 1
```

`RefCell<T>` is the general case. It hands out real references — `.borrow()`
for `&T`, `.borrow_mut()` for `&mut T` — and enforces the borrow rules at
runtime by tracking borrows in a counter. Break the rules and it **panics**
instead of failing to compile:

```rust
use std::cell::RefCell;

let cell = RefCell::new(vec![1, 2, 3]);
cell.borrow_mut().push(4);          // ok: one mutable borrow, then released
println!("{:?}", cell.borrow());    // prints: [1, 2, 3, 4]

let a = cell.borrow_mut();
let b = cell.borrow_mut();          // PANIC: already mutably borrowed
```

That's the whole deal with `RefCell`: it trades a compile error for a runtime
panic. You use it when you *know* your access pattern is safe but the borrow
checker can't prove it statically — typically because of shared ownership.

> **Trap:** A `RefCell` borrow guard lives until the end of its scope, not the
> end of the statement. `let x = cell.borrow_mut();` holds the borrow for the
> rest of the block, so a later `borrow()` panics. The fix is to keep guards
> short — drop them, or use the result inline — rather than binding them to
> long-lived locals.

### `Rc<RefCell<T>>`: the canonical shared-mutable pattern

Stack them and you get the workhorse single-threaded pattern: `Rc` for "many
owners," `RefCell` for "mutate through the shared reference." Several owners,
each able to mutate the one underlying value.

```rust
use std::rc::Rc;
use std::cell::RefCell;

let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
let alias = Rc::clone(&shared);      // two owners of the same RefCell

shared.borrow_mut().push(4);         // mutate through one owner...
println!("{:?}", alias.borrow());    // prints: [1, 2, 3, 4] — seen via the other
```

```go
// Go
shared := &[]int{1, 2, 3}
alias := shared               // a plain pointer everyone holds
*shared = append(*shared, 4)
fmt.Println(*alias)           // [1 2 3 4]
```

⚡ *Where the Go analogy breaks: in single-threaded Go this is just a pointer two
variables hold, with zero ceremony, because Go doesn't enforce aliasing rules at
all. `Rc<RefCell<T>>` is the price of getting that same shared-mutable behavior
in a language that does enforce them — `Rc` re-grants the sharing, `RefCell`
re-grants the mutation, and you've moved the borrow check from compile time to a
runtime counter. If your Go code shares a pointer and mutates it, this is the
Rust translation.*

---

## 7. `Arc<T>` + `Mutex<T>`: the thread-safe versions (forward ref)

Everything above is **single-threaded**. `Rc` uses a non-atomic counter and
`RefCell`'s borrow flag isn't synchronized, so neither is `Send`/`Sync` —
the compiler won't even let you move them across a thread boundary. The
thread-safe analogs are a one-to-one mapping: `Rc` → `Arc` (Atomic Reference
Counted), and `RefCell` → `Mutex` or `RwLock`. So the cross-thread cousin of
`Rc<RefCell<T>>` is `Arc<Mutex<T>>`. We cover those properly in Concurrency
(Part 9, `09-concurrency.md`) — for now just hold the mapping: same shapes,
atomic versions, real locks instead of a runtime borrow counter.

---

## 8. Reference cycles: the one place Rust leaks

`Rc` frees its value when the count hits zero. So what if two `Rc`s point at
each other? Each keeps the other's count at one, the count never reaches zero,
and the memory **leaks**. This is the one way to leak memory in safe Rust —
no `unsafe` required.

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node { next: RefCell<Option<Rc<Node>>> }

let a = Rc::new(Node { next: RefCell::new(None) });
let b = Rc::new(Node { next: RefCell::new(Some(Rc::clone(&a))) });
*a.next.borrow_mut() = Some(Rc::clone(&b)); // a -> b and b -> a: a cycle

// both counts stay at 1 forever; neither is ever freed — leaked
```

The fix is `Weak<T>`: a non-owning reference that does *not* bump the strong
count. A `Weak` doesn't keep its target alive; you call `.upgrade()` to try to
get an `Rc` back, which returns `None` if the target has already been freed. The
classic use is a parent/child tree where children own nothing upward: children
hold a strong `Rc` to nothing, the parent holds strong `Rc`s to children, and
each child holds a `Weak` back to its parent.

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    parent: RefCell<Weak<Node>>,    // Weak: does NOT keep the parent alive
    children: RefCell<Vec<Rc<Node>>>, // strong: parent owns its children
}

let leaf = Rc::new(Node {
    parent: RefCell::new(Weak::new()),
    children: RefCell::new(vec![]),
});
let branch = Rc::new(Node {
    parent: RefCell::new(Weak::new()),
    children: RefCell::new(vec![Rc::clone(&leaf)]),
});
*leaf.parent.borrow_mut() = Rc::downgrade(&branch); // weak link upward
// no cycle of strong refs -> everything frees correctly
```

```go
// Go
type Node struct {
    Parent   *Node   // a back-pointer — totally fine
    Children []*Node
}
// build whatever cycles you like; the tracing GC collects them anyway
```

⚡ *Where the Go analogy breaks: Go's garbage collector is **tracing**, not
reference-counting — it finds unreachable objects by walking from the roots, so
a cycle of pointers with no external reference is collected normally. You build
parent/child back-pointers in Go without a second thought. `Rc`'s
reference-counting can't see that a cycle is unreachable, so the `Weak` you write
here is you manually doing the job Go's GC did silently. This is one of the
clearest spots where Rust hands you a bill the GC was quietly paying.*

> **Trap:** Reference cycles compile and run fine — there's no error, no panic,
> just memory that's never reclaimed. If a long-running Rust program with `Rc`
> graphs grows without bound, suspect a cycle, and reach for `Weak` on the
> back-edges.

---

## 9. Choosing: a decision table

Start with plain ownership and borrowing. Reach for these only when a real
constraint forces you to.

| Situation                                              | Use                  |
| ----------------------------------------------------- | -------------------- |
| A value with one clear owner, passed/borrowed normally | plain `T` + `&T`/`&mut T` |
| Need the heap: recursion, large value, or `dyn Trait` | `Box<T>`             |
| Several owners, single-threaded, read-only             | `Rc<T>`              |
| Several owners, single-threaded, need to mutate        | `Rc<RefCell<T>>`     |
| Mutate a `Copy` field through a shared `&`             | `Cell<T>`            |
| Several owners across threads                          | `Arc<T>` (Part 9)    |
| Several owners across threads, mutable                 | `Arc<Mutex<T>>` (Part 9) |
| A back-edge that must not keep its target alive        | `Weak<T>`            |

The rule of thumb: each layer you add costs something — an allocation, an atomic
op, a runtime borrow check, a lock — so add the *minimum* that makes your shape
legal. If a borrow works, don't reach for `Rc`. If `Rc` works, don't reach for
`RefCell`. This is the inverse of the Go habit, where you'd just share a pointer
and let the GC and (if concurrent) a `sync.Mutex` sort it out; in Rust you name
exactly the sharing you need and pay for nothing else.

---

## One-sentence mental model

Smart pointers are the explicit, à-la-carte versions of what Go's GC and
free-sharing pointers do for free: `Box` heap-allocates with one owner, `Rc`/`Arc`
hand out shared ownership by counting references, `RefCell`/`Mutex` move the
borrow check to runtime so you can mutate shared data, `Drop` makes cleanup
deterministic instead of finalizer-roulette, and `Weak` breaks the reference
cycles that — unlike Go's tracing GC — a refcount can never reclaim.

---

[← Collections, iterators, closures](07-collections-iterators-closures.md) · [Next: Concurrency →](09-concurrency.md)
