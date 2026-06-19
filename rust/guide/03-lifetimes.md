# Part 3 — Lifetimes

[← Ownership and borrowing](02-ownership-and-borrowing.md) · [Next: Types and data →](04-types-and-data.md)

Lifetimes are the most intimidating-*looking* feature in Rust and one of the
simplest once you reframe them. A lifetime is not a thing that runs; it's a
**name for how long a borrow is valid**, used by the compiler to prove that no
reference ever outlives the data it points at — the dangling-reference rule from
Part 2, now made explicit. Go has no equivalent because its garbage collector
keeps pointed-to data alive for you; that contrast runs through this whole part.

---

## 1. The problem lifetimes solve

Part 2 ended with a borrow that can't dangle inside one function. Lifetimes
exist for the harder case: borrows that cross a function boundary, where the
compiler can no longer *see* both ends at once. Consider a function that returns
a reference picked from its arguments:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
// ERROR: missing lifetime specifier
//        this function's return type contains a borrowed value, but the
//        signature does not say whether it is borrowed from `x` or `y`
```

The body is obviously correct. The problem is the *signature*. The caller sees
only `fn longest(x: &str, y: &str) -> &str` — and from that alone, the compiler
checking the caller cannot tell how long the returned reference stays valid. Is
it tied to `x`? To `y`? To both? That answer decides whether the caller's code
is safe, so the compiler refuses to guess. You have to *name* the relationship.

In Go this function needs no thought at all:

```go
// Go
func longest(x, y string) string {
    if len(x) > len(y) {
        return x
    }
    return y
}
```

There is no dangling question to answer. A Go `string` is a header whose backing
bytes the GC keeps alive as long as *anything* references them, including the
returned value. The notion "how long is this borrow valid" has no place in Go,
because nothing is ever freed while still referenced. Lifetimes are the price of
deleting that GC — the static bookkeeping that replaces it.

---

## 2. Reading `'a` in a signature

Here is `longest` with the lifetime named. The annotation lives in angle
brackets, like a generic parameter, and `'a` (pronounced "tick-a") is the name:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

Read it left to right as a single sentence:

- `<'a>` — "for some lifetime `'a`,"
- `x: &'a str, y: &'a str` — "given two string references that are both valid
  for at least `'a`,"
- `-> &'a str` — "I return a string reference also valid for `'a`."

Because the return shares `'a` with both inputs, `'a` is forced to be the
*shorter* of the two input borrows — the overlap where both are still alive. The
practical meaning: **the returned reference lives no longer than the shorter of
its inputs.** You are not setting the lifetime; you are stating a constraint, and
the compiler solves for the actual region.

This is what makes the dangling case fail at the call site, exactly where the
bug would be:

```rust
let r;
{
    let s2 = String::from("short");
    let s1 = String::from("a longer string");
    r = longest(&s1, &s2);     // borrows tied to s1 and s2
    println!("{r}");           // fine: both still alive
}
// println!("{r}");            // ERROR: `s2` does not live long enough
                              //        borrowed value does not live long enough
```

`r` may point into `s2`, so `r` cannot outlive `s2`. The lifetime annotation is
what lets the compiler *know* that and reject the stale use. Without it, the
signature carried no such information.

> **Trap:** `'a` does not *change* how long anything lives. Annotations are
> descriptive, never prescriptive — you cannot extend a value's life by giving
> its reference a longer-named lifetime, any more than renaming a variable makes
> it live longer. If the data drops at `}`, no annotation saves it. Lifetimes
> only *describe* constraints the compiler then checks.

---

## 3. Lifetime elision: why you rarely write them

If every reference needed an explicit `'a`, Rust would be unbearable. It isn't,
because the compiler fills in the obvious cases for you. This is **lifetime
elision**, and it's why the overwhelming majority of functions that take and
return references carry *zero* lifetime annotations:

```rust
fn first_word(s: &str) -> &str {       // no 'a anywhere, yet returns a borrow
    s.split(' ').next().unwrap_or("")
}

fn len(s: &String) -> usize { s.len() } // borrow in, no borrow out — trivial
```

The compiler applies three mechanical rules to try to assign every lifetime
without your help:

1. **Each elided input reference gets its own distinct lifetime.** One `&` param
   gets `'a`, a second gets `'b`, and so on.
2. **If there is exactly one input lifetime, it is assigned to every elided
   output lifetime.** One reference in, the output borrow obviously comes from
   it — `first_word` resolves to `fn first_word<'a>(s: &'a str) -> &'a str`.
3. **If one of the inputs is `&self` or `&mut self`** (a method), the lifetime of
   `self` is assigned to every elided output. Methods returning a borrow almost
   always borrow from the receiver, so this covers most of them.

If those rules fully determine every output lifetime, you write nothing. If they
*don't* — as with `longest`, which has two input references and rule 2 can't pick
between them — the compiler stops and demands an explicit annotation. That demand
is the entire reason `longest` failed in §1.

⚡ *Where the Go analogy breaks: there is nothing to compare here, because Go has
no lifetimes to elide. The takeaway for a Go developer is the inverted ratio —
you will **read** lifetime annotations in other people's signatures far more
often than you will ever **write** one. Treat `<'a>` in a signature as
documentation the compiler verified, not as a hoop you must jump through daily.*

---

## 4. Lifetimes in structs that hold references

A struct field can be a reference. The moment it is, the struct itself must carry
a lifetime parameter, because the compiler has to know that the struct cannot
outlive the data its field borrows:

```rust
struct Parser<'a> {
    input: &'a str,    // borrows a string it does not own
    pos: usize,
}

impl<'a> Parser<'a> {
    fn new(input: &'a str) -> Parser<'a> {
        Parser { input, pos: 0 }
    }
    fn rest(&self) -> &str {       // elision rule 3: borrows from &self
        &self.input[self.pos..]
    }
}
```

Read `struct Parser<'a>` as: "a `Parser` is valid only for as long as the string
it borrows is valid." The compiler then enforces exactly that:

```rust
let parser;
{
    let text = String::from("1 + 2");
    parser = Parser::new(&text);   // parser borrows text
}                                  // text dropped here
// parser.rest();                  // ERROR: `text` does not live long enough
```

`parser` holds a `&'a str` into `text`; once `text` drops, `parser` would be a
struct full of dangling pointer, so any later use is rejected. The struct is
**tethered** to its borrowed data.

Go does the same thing structurally but asks nothing of you:

```go
// Go
type Parser struct {
    input string   // a header; the GC keeps the bytes alive
    pos   int
}

func main() {
    var p Parser
    {
        text := "1 + 2"
        p = Parser{input: text}   // p shares text's backing bytes
    }                             // text variable gone, bytes are NOT
    _ = p.input                   // perfectly fine — GC kept them alive
}
```

A Go struct field that's a pointer (or string, or slice) just *keeps its target
alive* — the field is itself a root the GC honors. So a Go struct can freely
outlive the local variable it was built from. A Rust struct holding a `&'a`
cannot: it borrows, it does not keep anything alive, and the `<'a>` is how the
compiler tracks the tether.

⚡ *Where the Go analogy breaks: in Go, putting a reference in a struct is a way
to **keep data alive**; in Rust, putting a `&'a` in a struct is a way to **borrow
data you promise to outlive**. If you actually want a Rust struct that owns its
string with no lifetime baggage, store an owned `String` instead of `&'a str` —
that's usually the right call, and it's the closest analog to what Go's struct
gives you for free.*

> **Trap:** reaching for `&'a str` fields to "avoid copying" is a classic early
> overreach. A struct with a lifetime parameter infects everything that holds
> it — every function taking one needs the lifetime threaded through, and the
> struct can never outlive its source. Start with owned fields (`String`,
> `Vec<T>`); switch to borrowed fields only when profiling or a real borrowing
> pattern (like a zero-copy parser over a buffer) demands it.

---

## 5. The `'static` lifetime

One lifetime has a name you'll see constantly: `'static`. It means "this
reference is valid for the entire duration of the program." The most common
source is string literals, which are baked into the binary:

```rust
let s: &'static str = "hello";   // lives in the program's static memory
```

Every string literal has type `&'static str` because the bytes live in the
binary's read-only data for the whole run — there is no scope they can outlive.
That is why a literal can be returned from any function with no lifetime worry:

```rust
fn greeting() -> &'static str {
    "hi"            // fine: the literal outlives every possible caller
}
```

`'static` also shows up as a *bound* (`T: 'static`), which is a different and
weaker claim — it means "this type contains no references that are shorter than
`'static`," i.e. the type could be held indefinitely. An owned `String` satisfies
`T: 'static` even though the string itself is dropped normally, because it
borrows nothing. You'll meet this bound when spawning threads (Part 9).

> **Trap:** `'static` does **not** mean "lives forever" or "leaks." A `&'static
> str` reference is allowed to be *used* briefly and forgotten — the name
> describes the maximum region it *could* be valid, not that anything is kept
> around. And `T: 'static` famously does **not** mean the value lives for the
> whole program; it means the type has no non-`'static` borrows in it. Owned data
> is `'static`-bound and still drops at the end of its scope like anything else.
> Misreading `'static` as "immortal" is the single most common lifetime
> confusion.

Go has nothing to annotate here, but the closest intuition is a Go string
literal or a package-level `const`/`var` string: it's just always there. The
difference is Go never makes you *say* so in a type.

---

## 6. When you actually must annotate

Most code needs no explicit lifetimes. The handful of situations that do are
worth memorizing, because they're the only times you'll reach for `'a`:

- **Returning a reference derived from multiple input references**, where
  elision rule 2 can't decide which input the output borrows from — the
  `longest` case (§2).
- **Structs (and enums) that hold references** — every borrowed field forces a
  lifetime parameter on the type (§4).
- **Some `impl` blocks and trait situations**, where you must restate the
  struct's lifetime: `impl<'a> Parser<'a> { ... }`, or thread a lifetime through
  a trait method's signature.
- Occasionally a **lifetime bound** like `<'a, T: 'a>` ("`T` must outlive `'a`"),
  which pins down that data behind a reference lives at least as long as the
  reference to it.

That's essentially the whole list for everyday Rust. If you find yourself writing
lifetimes in plain free functions that take one reference, the elision rules
almost certainly already handle it and the annotation is noise.

```go
// Go
// There is no analog to any bullet above. A Go function returning one of two
// input strings, a struct holding a *Foo, a method returning a field pointer —
// all of it compiles with zero annotation, because the GC, not the type system,
// guarantees the pointer stays valid. The cost is paid at runtime, not in syntax.
```

---

## 7. Why this feels like Rust is "fighting you"

For a Go developer, lifetimes are the #1 reason Rust feels combative early on.
You write a function, the body is plainly correct, and the compiler refuses it
over a borrow relationship you never had to think about in any GC'd language. It
feels like bureaucracy.

The reframe that makes it click: the borrow checker is catching **real bugs that
Go avoids only by always heap-allocating and leaning on the GC.** Go's answer to
"could this reference dangle?" is "it can't, because the GC keeps the target
alive as long as the reference exists" — which is correct, but it costs you a
runtime, GC pauses, and the occasional silent aliasing bug (the stale-slice case
from Part 2 §8). Rust's answer is "prove statically it can't dangle," and a
lifetime annotation is just you supplying the one fact the compiler couldn't
infer on its own.

Put bluntly: every lifetime error is a dangling-pointer bug the compiler found at
build time. In Go that same code shape is either fine (the GC saved you) or a
quiet correctness bug that ships. Rust converts a class of runtime
bugs into compile-time errors, and the annotations are the vocabulary that
conversion needs. Once you read `<'a>` as "the compiler proved this borrow is sound" rather
than "another hoop," the friction drops sharply.

⚡ *Where the Go analogy breaks: in Go you never restructure code to satisfy a
borrow checker, because there isn't one — you trade that for GC overhead and
runtime race detection. In Rust you will occasionally reshape code to make a
lifetime provable, and the honest tradeoff is real: you pay in up-front friction
to get zero-cost, guaranteed-sound references with no GC underneath.*

---

## 8. What's beyond this (and rarely needed early)

Lifetimes have advanced corners — **higher-ranked trait bounds** (`for<'a>
...`, mostly seen with closures and function pointers) and **variance** (how
lifetimes relate under subtyping) — but you can write substantial, idiomatic Rust
for a long time without ever invoking them by name; when you eventually need
them, the compiler error will point the way. Don't front-load them. The core
model in §1–§6 carries almost everything you'll do.

---

## One-sentence mental model

A lifetime is a compiler-checked *name* for the region during which a borrow
stays valid, written `'a` and threaded through signatures and struct definitions
so the compiler can prove no reference ever outlives its data — the same
dangling-pointer guarantee Go buys at runtime with its garbage collector, except
Rust buys it at compile time with annotations you mostly read rather than write.

---

[← Ownership and borrowing](02-ownership-and-borrowing.md) · [Next: Types and data →](04-types-and-data.md)
