# Unsafe and FFI

*A sibling deep-dive to [the Go developer's guide to Rust](guide/00-overview.md).
Natural follow-up to [Part 8](guide/08-smart-pointers-and-interior-mutability.md).*

`unsafe` is the most misunderstood keyword in Rust. Coming from Go, it's
tempting to read it as "turn the safety off, go full C." It does not do that.
`unsafe` unlocks exactly five extra abilities and nothing else — the borrow
checker, the type checker, ownership, and lifetimes all still apply, inside the
block as much as outside it. This deep-dive is about what `unsafe` actually buys
you, the discipline around it, and Rust's standout use case for it: calling C
with essentially zero overhead.

---

## 1. What `unsafe` actually unlocks

`unsafe` is not a mode that loosens the rules. It's a marker that says: "the
compiler can't prove this particular operation is sound, so I, the author, am
asserting it is." It enables precisely **five** extra abilities:

1. **Dereference a raw pointer** (`*const T` / `*mut T`).
2. **Call an `unsafe` function** — including all foreign (FFI) functions.
3. **Access or modify a mutable `static`** variable.
4. **Implement an `unsafe` trait** (e.g. `Send` / `Sync` by hand).
5. **Access the fields of a `union`.**

That's the whole list. Everything else you know still holds. You cannot alias a
`&mut` past the borrow checker, you cannot ignore lifetimes, you cannot leak a
move — none of that changes inside `unsafe {}`. The block delimits *where* the
five superpowers are allowed, and equally importantly, *where a reviewer should
look hard*.

> **Trap:** `unsafe` does **not** disable the borrow checker. If you write code
> that aliases a mutable reference, it's rejected whether or not it's wrapped in
> `unsafe`. The five powers above are additive; nothing is subtracted.

```rust
unsafe {
    let mut x = 5;
    let r1 = &mut x;
    let r2 = &mut x; // STILL an error: cannot borrow `x` mutably twice
    *r1 += 1;
    *r2 += 1;
}
```

Compare Go's `unsafe`, which is closer in spirit but blunter:

```go
// Go
import "unsafe"

p := unsafe.Pointer(&x)
// You can now reinterpret memory, do uintptr arithmetic, bypass the type
// system entirely. There is no scoped block; once you import unsafe, any line
// in the file can do these things, and you own all the consequences.
```

Both languages say "you take responsibility here." The difference is *scope*:
Go's `unsafe` colours a whole package's intent loosely, while Rust's `unsafe {}`
brackets the exact expressions that need auditing.

---

## 2. What `unsafe` *means* — a contract, not an escape hatch

The mental reframe: `unsafe` is a **promise you make to the compiler**, not a
permission the compiler grants you. The compiler normally proves soundness for
you. When it can't (a raw pointer might be null, a C function might do anything),
it refuses — unless you wrap the operation in `unsafe` and thereby sign off on it
yourself.

This means an `unsafe` block is a *liability marker*. In a code review you grep
for `unsafe` and scrutinise every one. In safe Rust, a memory bug can only
originate inside an `unsafe` block somewhere in the dependency tree — that's the
entire value proposition. Safe code stays safe; the audit surface shrinks to the
bracketed regions.

⚡ *Where the Go analogy breaks: in Go, the absence of `unsafe` doesn't prove the
absence of data races — `-race` is a runtime probe, not a proof. In Rust, if
there's no `unsafe` (yours or a dependency's), there is no UB, full stop. The
keyword turns "trust the whole program" into "trust these N small blocks."*

---

## 3. Raw pointers: `*const T` and `*mut T`

References (`&T`, `&mut T`) are checked: non-null, never dangling, lifetime-bound,
aliasing-controlled. Raw pointers throw all of that away. A `*const T` or
`*mut T`:

- is **not** borrow-checked,
- can be **null**,
- can **dangle** (point at freed memory),
- carries **no lifetime**,
- can be freely aliased (many `*mut` to the same place is fine to *hold*).

Crucially: **creating** a raw pointer is safe. Only **dereferencing** it needs
`unsafe`, because that's the moment the unverifiable promise is made.

```rust
let mut n = 42;
let p_const = &n as *const i32; // creating: safe
let p_mut   = &mut n as *mut i32; // creating: safe

unsafe {
    println!("{}", *p_const); // deref: unsafe. prints: 42
    *p_mut = 99;              // deref: unsafe
}
println!("{}", n);            // prints: 99

// You can even make a guaranteed-dangling pointer safely; just never deref it:
let dangling = 0x1234 as *const i32; // safe to construct, UB to read
```

```go
// Go
p := &n                 // *int — a real, GC-tracked pointer
// Go pointers can't do arithmetic and are never dangling (the GC keeps the
// pointee alive). To get C-like raw behaviour you must go through
// unsafe.Pointer and uintptr:
up := uintptr(unsafe.Pointer(&n)) + 8   // pointer arithmetic, your problem now
```

The Go takeaway: an ordinary Go `*int` is much closer to Rust's `&` (managed,
safe) than to Rust's `*mut` (raw). Rust's raw pointers are the analogue of
`unsafe.Pointer`/`uintptr`, and you reach for them roughly as rarely.

> **Trap:** A raw pointer has no lifetime, so the compiler won't stop you from
> holding one past the death of what it points to. Keeping a `*mut T` alive after
> its owner drops is a use-after-free waiting to happen — and now it's entirely on
> you, exactly as in C.

---

## 4. The discipline: wrap `unsafe` in a safe abstraction

The cultural rule that makes this work: **minimise and encapsulate**. You don't
sprinkle `unsafe` through application code. You write a small, carefully-audited
`unsafe` core that upholds an invariant, then expose a **safe** API that the rest
of the program — and the borrow checker — can rely on.

The standard library is built this way. `Vec<T>` is raw-pointer manipulation all
the way down, yet its public surface is completely safe. A cleaner example is
`split_at_mut`, which hands out two non-overlapping `&mut` slices from one — the
borrow checker can't see they don't overlap, so the implementation uses `unsafe`
to assert it:

```rust
fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();
    assert!(mid <= len); // the invariant that makes the unsafe sound
    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
// Callers see two safe &mut slices. The unsafe is sealed behind the assert.
```

The `assert!` is load-bearing: it's the proof obligation the `unsafe` block
depends on. A good `unsafe` block is always paired with the reasoning (often a
`// SAFETY:` comment) for *why* the promise holds. This is the same instinct
behind the smart pointers in [Part 8](guide/08-smart-pointers-and-interior-mutability.md):
`Rc`, `RefCell`, and `Mutex` are safe wrappers around `unsafe` interiors.

---

## 5. FFI: calling C from Rust

This is where `unsafe` earns its keep. You declare foreign functions in an
`extern "C"` block; the `"C"` is the ABI (calling convention). The declarations
are just signatures — calling one is `unsafe`, because Rust can't verify what C
does.

Calling `abs` from the C standard library:

```rust
extern "C" {
    fn abs(input: i32) -> i32; // declared, not defined — libc provides it
}

fn main() {
    let n = unsafe { abs(-7) }; // the call site is unsafe
    println!("{n}"); // prints: 7
}
```

For a function from a named library, attach `#[link(...)]`:

```rust
#[link(name = "m")] // link libm
extern "C" {
    fn sqrt(x: f64) -> f64;
}
```

When you pass structs across the boundary, lay them out the way C expects with
`#[repr(C)]` — Rust's default layout is unspecified and free to reorder fields:

```rust
#[repr(C)]
struct Point {
    x: f64,
    y: f64,
}

extern "C" {
    fn distance(a: *const Point, b: *const Point) -> f64;
}
```

The matching C side:

```c
typedef struct { double x, y; } Point;

double distance(const Point *a, const Point *b) {
    double dx = a->x - b->x, dy = a->y - b->y;
    return /* sqrt */ (dx*dx + dy*dy);
}
```

Across the boundary you pass raw pointers, and **you manage memory by hand**.
There's no GC and no `Drop` running on the C side — if C `malloc`s something, you
free it through C; if Rust owns something, Rust drops it. Mixing them up is a
leak or a double-free.

Now the comparison that matters:

```go
// Go — cgo
/*
#include <stdlib.h>
double distance(...);
*/
import "C"

d := C.distance(a, b)
// cgo works, but every call crosses a real runtime boundary: the Go scheduler
// must hand the goroutine off to an OS thread that can run C, pointers passed
// in are subject to cgo's pointer-passing rules (the GC must not move/collect
// them), and there's measurable per-call overhead. cgo calls are famously
// "not free" — people avoid them in hot loops.
```

This is a genuine, structural Rust advantage. **Rust has no runtime and no GC**,
so an `extern "C"` call is essentially a normal function call — close to
zero overhead, no scheduler handoff, no pointer-pinning dance. Where cgo makes
you think twice about call frequency, Rust FFI you can put in a tight loop.

⚡ *Where the Go analogy breaks: cgo's cost comes from the Go runtime needing to
protect its scheduler and moving GC from C's interference. Rust has neither, so
the "boundary" largely evaporates — C interop is one of the things Rust is
flatly better at than Go.*

---

## 6. Exposing Rust *to* C

The reverse direction: mark a function `#[no_mangle]` (so the symbol name isn't
mangled) and `pub extern "C"` (C ABI), then build a C-compatible library.

```rust
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Tell Cargo to emit a C library in `Cargo.toml`:

```toml
[lib]
crate-type = ["cdylib"]   # a .so/.dll/.dylib; use "staticlib" for a .a
```

Now any C, Python (via ctypes), or Go (via cgo) program can call `add` as if it
were a C function. This is a common way to ship a fast Rust core into an existing
codebase incrementally.

> **Trap:** Anything reachable across an `extern "C"` boundary must be
> FFI-safe — `#[repr(C)]` types, primitives, raw pointers. You can't hand C a
> Rust `String`, `Vec`, `Option`, or trait object and expect it to make sense;
> their layouts are Rust's private business. Convert to C-shaped types at the
> edge.

---

## 7. The tooling: `bindgen` and `cc`

Two crates do the tedious parts so you rarely hand-write the boilerplate above:

- **`bindgen`** reads a C header and auto-generates the Rust `extern` declarations
  and `#[repr(C)]` structs for you.
- **`cc`** is a build-script helper that compiles a bundled `.c` file as part of
  `cargo build`, so you can ship C source alongside your crate.

Together they're the standard path for wrapping a C library: `cc` builds it,
`bindgen` binds it, and you write a thin safe wrapper on top (Section 4).

---

## 8. When you actually need `unsafe`

Almost never, in application code. If you're writing a web service, a CLI, or
business logic, you can go your whole Rust career without typing `unsafe` once —
the safe standard library and ecosystem already wrap everything you need. Where
it legitimately shows up:

- **FFI** — binding to C/system libraries (the most common reason).
- **Low-level data structures** — lock-free queues, arenas, custom allocators,
  anything where you're building the abstraction others rely on.
- **Performance-critical hot paths** — eliding a bounds check you've already
  proven, after profiling says it matters.

If you find yourself reaching for `unsafe` to "make the borrow checker stop
complaining," that's a smell — the answer is almost always a different ownership
design (revisit [Part 2](guide/02-ownership-and-borrowing.md)) or one of the smart
pointers from [Part 8](guide/08-smart-pointers-and-interior-mutability.md), not a
raw pointer. `unsafe` is a tool for *building* safe abstractions, not for *opting
out* of them.

---

## One-sentence mental model

`unsafe` doesn't switch off Rust's safety — it grants exactly five extra powers
in a bracketed, auditable region where you, not the compiler, vouch for
soundness, and its sweet spot is near-zero-overhead C interop that Go's cgo can't
match.

---

[← Back to the guide](guide/00-overview.md)
