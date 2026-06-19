# Macros

*A sibling deep-dive to [the Go developer's guide to Rust](guide/00-overview.md).
Natural follow-up to [Part 1](guide/01-reading-rust.md) and
[Part 5](guide/05-traits-and-generics.md).*

You hit `println!` on day one and the `!` looked like a typo. It isn't — it's
the single most visible piece of a whole language feature Go simply does not
have: macros. This deep-dive answers "why the `!`," shows you how to *read* a
macro (you'll read thousands before you write one), and maps each kind onto the
closest Go thing — which, more often than not, is `go generate`.

---

## 1. The `!` means "this is a macro, not a function"

In Rust, `name(...)` is a function call and `name!(...)` is a *macro
invocation*. The `!` is mandatory syntax that flags: this runs at **compile
time** and expands into real code before the compiler ever type-checks your
program. `println!`, `vec!`, `format!`, `assert_eq!` — all macros.

```rust
fn main() {
    let name = "world";
    println!("hello, {name}");        // macro: expands at compile time
    let v = vec![1, 2, 3];            // macro: expands to Vec-building code
    let s = format!("{}-{}", 1, 2);   // macro: returns a String
    println!("{v:?} {s}");            // prints: [1, 2, 3] 1-2
}
```

Go has nothing here. `fmt.Println` is an ordinary function, full stop.

```go
// Go
func main() {
    name := "world"
    fmt.Printf("hello, %s\n", name) // plain function call, no codegen
    v := []int{1, 2, 3}             // literal, not a macro
}
```

That difference is not cosmetic. Because `println!` is a macro, it reads your
format string **at compile time** and verifies the arguments match:

```rust
println!("{} {}", 1);   // compile error: 2 placeholders, 1 argument
```

```go
// Go
fmt.Printf("%d %d", 1)  // COMPILES FINE — blows up or misprints at runtime
// `go vet` can catch some of these, but the compiler itself never does.
```

This is the concrete, day-one win to internalize. Go's `fmt` package is a plain
function, so it can't see your format string until runtime — which is exactly
why it leans on `reflect` and `interface{}` to inspect each argument's type as
the program runs. Rust's `println!` does that work during compilation and emits
straight-line, reflection-free code.

> **Trap:** `!` is not the boolean NOT operator here. `!x` negates a bool;
> `x!(...)` invokes a macro. Same character, unrelated meanings — the trailing
> `(`, `[`, or `{` is what tells them apart.

---

## 2. Why macros exist at all

Rust has macros to do things ordinary functions provably cannot:

- **Variadic arguments.** Rust functions have a fixed arity — there is no `...`
  like Go's `func(args ...int)`. `vec![1, 2, 3]`, `vec![1, 2, 3, 4]`, and
  `println!` with any number of args all work because macros, not functions,
  accept a variable token stream.
- **Compile-time codegen.** Generate an `impl` block, a builder, a parser — at
  compile time, type-aware.
- **DSLs.** Embed little sub-languages: SQL in `sqlx::query!`, HTML in many web
  frameworks, format strings in `format!`.
- **Killing boilerplate.** `#[derive(Debug)]` writes a `Debug` impl you'd
  otherwise type by hand.

The everyday ones you'll use constantly:

```rust
let v = vec![0u8; 1024];            // a 1024-byte buffer
let msg = format!("{x} of {y}");    // build a String
assert_eq!(got, want);             // test assertion w/ nice diff on failure
dbg!(&v);                          // print "[src/main.rs:3] &v = [...]" + return v
todo!();                           // compiles, panics at runtime: "not yet implemented"
unimplemented!();                  // like todo!, "semantically not implemented"
panic!("boom: {code}");            // abort with a message
```

`todo!()` deserves a Go callout. In Go you'd stub a function with
`panic("TODO")` or `return nil, errors.New("unimplemented")` and the compiler
would still demand you satisfy the return type. `todo!()` has type `!` (the
"never" type), so it slots into *any* position and type-checks — let you sketch
signatures and fill bodies later without fighting return types.

⚡ *Where the Go analogy breaks: Go's answer to "I need variadic + type-checked
formatting" is reflection at runtime. Rust's answer is a macro at compile time.
There is no runtime reflection involved in `println!` at all.*

---

## 3. Declarative macros: `macro_rules!` (how to READ one)

There are two families. **Declarative** macros (`macro_rules!`) are pattern-match
templates: "when the input looks like *this*, emit *that*." **Procedural** macros
(§4) are compiled Rust programs. Start with declarative — most macros you'll read
in app code are these.

Here's a simplified `vec!`-style macro. Read it as a `match` over *syntax*:

```rust
macro_rules! myvec {
    // arm 1: myvec![] -> empty Vec
    () => {
        Vec::new()
    };
    // arm 2: one or more comma-separated expressions
    ( $( $x:expr ),* $(,)? ) => {
        {
            let mut v = Vec::new();
            $( v.push($x); )*
            v
        }
    };
}

fn main() {
    let a: Vec<i32> = myvec![];
    let b = myvec![1, 2, 3];
    println!("{a:?} {b:?}"); // prints: [] [1, 2, 3]
}
```

Decoding the pieces:

- Each `... => { ... }` is an **arm**: a matcher on the left, a template on the
  right. The macro tries arms top-to-bottom, like a `match`.
- `$x:expr` is a **metavariable** named `x` that must match a Rust *expression*.
  The `:expr` is a *fragment specifier*. Common ones: `expr`, `ident` (a name),
  `ty` (a type), `tt` (a single token tree), `pat`, `block`, `literal`.
- `$( ... ),*` is **repetition**: "zero or more, separated by commas." Inside the
  template, `$( v.push($x); )*` repeats the body once per matched `$x`. The `*`
  means zero-or-more; `+` means one-or-more; `?` means zero-or-one. The `$(,)?`
  at the end just allows an optional trailing comma.

So `myvec![1, 2, 3]` **expands to**:

```rust
// expands to:
{
    let mut v = Vec::new();
    v.push(1); v.push(2); v.push(3);
    v
}
```

A `hashmap!` macro follows the same shape with a `key => value` matcher:

```rust
macro_rules! hashmap {
    ( $( $k:expr => $v:expr ),* $(,)? ) => {{
        let mut m = std::collections::HashMap::new();
        $( m.insert($k, $v); )*
        m
    }};
}

// hashmap!{ "a" => 1, "b" => 2 } expands to insert() calls, returns a HashMap
```

The Go contrast is stark: there is no construct that pattern-matches over source
syntax and emits code. The nearest thing is a text/template file fed to
`go generate`, but that operates on strings, not parsed syntax, and is utterly
ignorant of Rust's types and scoping.

**Hygiene** is the one safety property to know. Macro-introduced variables don't
collide with or leak into your code, even if names clash:

```rust
macro_rules! sneaky {
    () => { let x = 999; };
}

fn main() {
    let x = 1;
    sneaky!();          // introduces its OWN x, not yours
    println!("{x}");    // prints: 1   (your x is untouched)
}
```

In a C macro this would clobber your `x`. Rust macros are *hygienic*: each
expansion gets its own syntactic context. You can read macro code without fear
that it's silently rebinding your locals.

> **Trap:** the matcher syntax is *not* regular Rust. `$x:expr`, `$(...)* `, and
> the fragment specifiers only exist inside `macro_rules!`. Don't try to read the
> left side of `=>` as normal code — read it as a grammar pattern.

---

## 4. Procedural macros: the three kinds

Procedural ("proc") macros are a different beast: they're **compiled Rust
programs** that receive your code as a stream of tokens and return a new stream
of tokens. Far more powerful than `macro_rules!` — they can parse, inspect types,
and generate arbitrary code. They live in their own crate and run as a compiler
plugin. There are three kinds.

### Derive macros — `#[derive(...)]`

By far the most common. `#[derive(Debug)]` on a struct *generates an `impl`
block* for you — the exact boilerplate you met in Part 4 and Part 5.

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point { x: i32, y: i32 }

// #[derive(Debug)] expands to roughly:
// impl std::fmt::Debug for Point {
//     fn fmt(&self, f: &mut Formatter) -> fmt::Result {
//         f.debug_struct("Point").field("x", &self.x).field("y", &self.y).finish()
//     }
// }
```

`#[derive(Serialize, Deserialize)]` from `serde` is the headline example: it
reads your struct's fields at compile time and writes a full, type-checked
JSON (de)serializer.

```go
// Go
type Point struct {
    X int `json:"x"`
    Y int `json:"y"`
}
// encoding/json reads those struct tags at RUNTIME via reflection.
// serde's derive does the equivalent at COMPILE time and emits direct
// field-access code — no reflection, and a type error is a build error.
```

That's the recurring Go-breaks moment: Go reaches for reflection and struct tags
*at runtime*; Rust reaches for a derive macro *at compile time*.

### Attribute macros — `#[name]`

These attach to an item (function, struct, module) and *transform* it. `#[test]`
turns a function into a test harness entry; `#[tokio::main]` rewrites your `async
fn main` into a synchronous `main` that spins up a runtime.

```rust
#[tokio::main]
async fn main() {
    // ...
}
// expands to roughly:
// fn main() {
//     tokio::runtime::Runtime::new().unwrap().block_on(async { /* body */ })
// }
```

### Function-like proc macros — `name!(...)`

Look like `macro_rules!` calls but are backed by a full program. `sqlx::query!`
is the showcase: it parses the SQL string, *connects to your database at compile
time*, and verifies the query and column types against the real schema.

```rust
let row = sqlx::query!("SELECT name, age FROM users WHERE id = $1", id)
    .fetch_one(&pool)
    .await?;
// `row.name` and `row.age` are typed from the DB schema, checked at build time.
```

The closest Go analog to all three is `go generate` plus a codegen tool
(`stringer`, `mockgen`, `sqlc`). But proc macros are *integrated* into the build
and *type-aware* — no separate step, no checked-in generated files, no
out-of-date `_gen.go`. The cost is that the magic is invisible (§6).

---

## 5. When you'll WRITE vs just USE

Calibrate your expectations:

- **Using** macros is constant and unremarkable: `println!`, `vec!`, `format!`,
  `assert_eq!`, `#[derive(...)]`, `#[tokio::main]`, `#[test]`. You'll do this
  every day without thinking of it as "doing macros."
- **Writing** a small `macro_rules!` happens occasionally in app code — to kill a
  repetitive pattern the type system can't abstract over (e.g. generating similar
  test cases). Reach for it sparingly; prefer a plain function or generic first.
- **Writing proc macros** is a library-author activity. `serde`, `tokio`, `clap`,
  and `sqlx` ship them; most app developers never write one. If you find yourself
  reaching for it, ask whether a trait + generics (Part 5) does the job instead.

So the realistic skill order is: *use* fluently, *read* `macro_rules!` when
debugging, *write* the occasional small declarative macro, and treat proc-macro
authoring as advanced/optional.

---

## 6. Costs and cautions (the honest part)

Macros are not free:

- **Compile times.** Heavy proc-macro use (large `serde` structs, `sqlx`)
  measurably slows builds. The codegen happens every compile.
- **Obscured errors.** A type error *inside* an expansion can point at the macro
  invocation, not the real culprit. Error messages from deep within a generated
  `serde` impl can be cryptic.
- **Tooling gaps.** Go-to-definition, autocomplete, and refactoring tools see
  generated code less reliably than hand-written code, though `rust-analyzer`
  keeps improving here.

> **Trap:** don't reach for a macro when a function or generic will do. Macros
> defeat type inference at the call boundary and make code harder for both humans
> and tools to follow. The Rust community norm is "macros are a last resort,"
> not a first one.

The learning tool that makes all of this concrete is **`cargo expand`** (install
with `cargo install cargo-expand`). It prints your code with every macro fully
expanded — so you can *see* what `#[derive(Debug)]` or `println!` actually
becomes. When a macro confuses you, expand it and read the plain Rust
underneath. It's the fastest way to demystify the `!`.

---

## One-sentence mental model

A macro is the compile-time code generator Go does with `go generate` and
reflection — except built into the language, type-aware, and marked at every call
site by a `!`, `#[derive(...)]`, or `#[attr]`.

---

[← Back to the guide](guide/00-overview.md)
