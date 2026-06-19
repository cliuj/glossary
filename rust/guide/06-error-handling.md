# Part 6 — Error handling

[← Traits and generics](05-traits-and-generics.md) · [Next: Collections, iterators, closures →](07-collections-iterators-closures.md)

This is the most direct Rust-vs-Go contrast in the whole guide. Go's `if err
!= nil` and Rust's `Result` + `?` are solving the *same* problem — errors are
ordinary values you must deal with, not exceptions thrown past you — and they
even agree on the philosophy. The difference is purely mechanical: Go makes you
write the propagation by hand every time; Rust gives you a one-character
operator that does it for you and converts the error type on the way out. Get
that one operator and you've got most of this Part.

---

## 1. Recap: `Result<T, E>` is the foundation

From Part 4 (`04-types-and-data.md` §6), `Result` is just an enum:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

A function that can fail returns a value that is *either* a success payload
`Ok(T)` *or* a failure payload `Err(E)`. There is no third channel — no
exceptions for recoverable errors, no `throw`, no stack-unwinding control flow
that skips your code. If a function can fail, its return type says so, and you
cannot get at the `T` without acknowledging the `E` might be there instead.

```rust
fn parse_port(s: &str) -> Result<u16, std::num::ParseIntError> {
    s.parse::<u16>() // parse() returns Result; we just hand it back
}
```

```go
// Go
func parsePort(s string) (uint16, error) {
    n, err := strconv.ParseUint(s, 10, 16)
    return uint16(n), err
}
```

This is the same contract Go has built its whole error story on: errors are
values, returned alongside (Go) or instead of (Rust) the result, and the caller
decides what to do. If you're comfortable with `(val, err)`, you already
understand the *model*. The rest of this Part is ergonomics and types.

⚡ *Where the Go analogy breaks: Go's `(val, err)` is a pair where both slots
always exist — on success `err` is `nil`, on failure `val` is the zero value,
and nothing stops you from reading a meaningless `val` after an error. Rust's
`Result` is a sum type: exactly one slot exists. You literally cannot touch the
success value while it's in the `Err` state, because it isn't there.*

---

## 2. The Go baseline, shown honestly

Let's be fair to Go first, because its approach is genuinely fine — explicit,
greppable, no hidden control flow. Here's a function that reads a config file,
parses a port out of it, and validates it. In Go it's the familiar staircase:

```go
// Go
func loadPort(path string) (uint16, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return 0, err
    }
    port, err := strconv.ParseUint(strings.TrimSpace(string(data)), 10, 16)
    if err != nil {
        return 0, err
    }
    if port == 0 {
        return 0, fmt.Errorf("port must be non-zero")
    }
    return uint16(port), nil
}
```

Three potential failures, three `if err != nil` blocks. It's verbose, but it's
honest: every error path is visible on the page. Nobody who's written Go for a
week finds this surprising. The complaint isn't that it's *wrong* — it's that
the happy path is buried under boilerplate, and the boilerplate is identical
every time.

---

## 3. The `?` operator — the headline

Here is the same logic in Rust, written the verbose way first so the `match`
maps cleanly onto Go's `if err != nil`:

```rust
use std::fs;

fn load_port_verbose(path: &str) -> Result<u16, Box<dyn std::error::Error>> {
    let data = match fs::read_to_string(path) {
        Ok(d) => d,
        Err(e) => return Err(Box::new(e)),
    };
    let port = match data.trim().parse::<u16>() {
        Ok(p) => p,
        Err(e) => return Err(Box::new(e)),
    };
    if port == 0 {
        return Err("port must be non-zero".into());
    }
    Ok(port)
}
```

Every `match … return Err(e)` is *exactly* Go's `if err != nil { return ..., err
}`. Same shape, same explicitness, same tedium. Now the `?` version:

```rust
use std::fs;

fn load_port(path: &str) -> Result<u16, Box<dyn std::error::Error>> {
    let data = fs::read_to_string(path)?;
    let port = data.trim().parse::<u16>()?;
    if port == 0 {
        return Err("port must be non-zero".into());
    }
    Ok(port)
}
```

That `?` is the whole story. On an expression of type `Result<T, E>`:

- if it's `Ok(v)`, `?` evaluates to `v` and execution continues;
- if it's `Err(e)`, `?` **returns early** from the enclosing function with
  `Err(e)` — after converting `e` into the function's declared error type.

So `let data = fs::read_to_string(path)?;` means "unwrap the success, or bubble
the error up to my caller." It's `if err != nil { return err }` compressed to
one character, applied automatically.

```go
// Go — the ? operator is literally this, generated for you:
data, err := os.ReadFile(path)
if err != nil {
    return nil, err
}
```

> **Trap:** `?` only works inside a function whose return type can absorb the
> error — i.e. a function returning `Result<_, E>` (or `Option<_>`, see §4). You
> cannot sprinkle `?` in a `fn` that returns a plain `u16`; the compiler will
> tell you `the ? operator can only be used in a function that returns Result`.
> There's no equivalent restriction in Go because Go has no `?` — but the mental
> model is the same: you can only "return the error" from somewhere that returns
> an error.

### The auto-conversion: `?` calls `From`

The "after converting" clause above is the part with no Go analogy. When `?`
bubbles an `Err(e)`, it runs `From::from(e)` to turn the inner error into the
function's declared error type (`From`/`Into` are from Part 5,
`05-traits-and-generics.md`). In `load_port`, `read_to_string` yields
`std::io::Error` and `parse` yields `ParseIntError` — two *different* types —
yet both `?` lines compile, because there's a `From<io::Error>` and a
`From<ParseIntError>` for `Box<dyn Error>`. The operator silently widens each
specific error into the common one.

⚡ *Where the Go analogy breaks: in Go, propagating two different error types
through one function "just works" because `error` is a single interface and
every error already satisfies it — there's no conversion step. Rust's errors are
concrete types by default, so `?` needs the `From` impl to unify them. The
upside is that once you define your own error enum (§7) with `From` impls, `?`
converts *into your domain type* automatically — something Go's untyped `error`
can't express without manual `fmt.Errorf` wrapping.*

---

## 4. `?` on `Option` too

`?` isn't `Result`-only. On an `Option<T>`, it unwraps `Some(v)` or
early-returns `None`. This is the "short-circuit on absence" pattern:

```rust
fn first_char_upper(s: &str) -> Option<char> {
    let c = s.chars().next()?;     // None here -> whole fn returns None
    Some(c.to_ascii_uppercase())
}
```

```go
// Go — no ? for the "ok" pattern, so you branch by hand:
func firstCharUpper(s string) (rune, bool) {
    if len(s) == 0 {
        return 0, false
    }
    r := []rune(s)[0]
    return unicode.ToUpper(r), true
}
```

A function using `?` on `Option` must itself return `Option` (or another type
implementing the right `Try` machinery). You can't mix: `?` on an `Option`
inside a `Result`-returning function won't compile without an explicit bridge
like `.ok_or(...)` (see §10). Keep the channels straight — `Option` for
"nothing," `Result` for "failed, and here's why."

---

## 5. Propagate or handle: when to `?` vs `match`

`?` is for *propagation* — "I can't deal with this, my caller might." But not
every error should bubble. Sometimes the current function is the right place to
handle it. That's Go's exact "handle here or return up" decision, just with
different syntax for each branch.

Propagate (bubble up) with `?`:

```rust
let config = load_config()?; // caller deals with a bad config
```

Handle locally with `match` or `if let` — e.g. fall back to a default when a
file is missing, but still fail on a real I/O error:

```rust
use std::io;

let port = match load_port("config.txt") {
    Ok(p) => p,
    Err(e) if is_not_found(&e) => 8080, // missing file -> default
    Err(e) => return Err(e),            // anything else -> propagate
};
```

```go
// Go — same fork: recover from one case, propagate the rest
port, err := loadPort("config.txt")
if errors.Is(err, os.ErrNotExist) {
    port = 8080
} else if err != nil {
    return err
}
```

Rule of thumb, identical in both languages: `?` (or `return err`) when you have
nothing useful to add and the caller knows more than you; `match`/`if let` when
*this* layer is where the decision actually belongs.

---

## 6. `unwrap`, `expect`, and `panic!`

Sometimes you want to say "this cannot fail, and if it does the program is
broken." That's what `.unwrap()` and `.expect(msg)` are for. They pull the `Ok`
/ `Some` value out, and if it's `Err` / `None` they **panic** — abort the
current thread with a message:

```rust
let port: u16 = "8080".parse().unwrap();           // fine: literal is valid
let home = std::env::var("HOME").expect("HOME unset"); // documents the assumption
```

`unwrap` panics with a generic message; `expect` lets you state the invariant
you're relying on, which becomes the panic text. **Prefer `expect`** — the
message is free documentation for the next person (often you).

When are these acceptable?

- **Prototypes / scripts / `main` glue** where crashing is an okay failure mode.
- **Genuine invariants** the type system can't see — e.g. you just inserted a
  key, so `map.get(&k).unwrap()` truly can't fail.
- **Tests**, where a panic *is* the failure report you want.

When are they a code smell? Anywhere a user-supplied or I/O value reaches
`unwrap` — that's a recoverable error masquerading as a crash. In real code
paths, return a `Result` and `?` it instead.

`panic!` itself is the explicit version, plus it's what fires on bugs like array
out-of-bounds indexing and (in debug builds) integer overflow:

```rust
let v = vec![1, 2, 3];
let x = v[10];          // panics: index out of bounds
panic!("unreachable: bad state {:?}", v); // explicit, for impossible cases
```

```go
// Go — this is panic(), and the analogy is tight:
xs := []int{1, 2, 3}
_ = xs[10]              // panics: index out of range
panic(fmt.Sprintf("unreachable: bad state %v", xs))
```

So the mapping is clean: Rust `panic!`/`unwrap` ≈ Go `panic`. Both are for
**unrecoverable bugs and invariant violations, not control flow.** You do not
build error handling out of panics any more than you'd build it out of Go's
`panic`.

> **Trap:** by default a Rust panic *unwinds* the stack (running destructors),
> and `std::panic::catch_unwind` can intercept it — which looks temptingly like
> Go's `recover()`. Don't. `catch_unwind` exists for FFI boundaries and thread
> isolation, not for `try/catch`-style error handling. Using it as flow control
> is as wrong in Rust as wrapping every Go function in `defer recover()`. If a
> value can legitimately be absent or a call can legitimately fail, that's a
> `Result`, not a panic to be caught.

⚡ *Where the Go analogy breaks: Go's `recover()` is a normal, if discouraged,
tool — HTTP servers routinely recover per-request so one bad handler doesn't
kill the process. Rust's culture treats catching panics as exotic. The
equivalent "don't crash the whole server on one bad request" job is done by
returning `Result` from handlers (the framework turns `Err` into a 500), and by
the OS/thread boundary, not by routinely catching unwinds.*

---

## 7. Designing your own error type

`Box<dyn Error>` (next section) is great for apps, but a *library* usually wants
a precise, matchable error type so callers can react to specific cases. The
idiomatic shape is an enum — one variant per failure mode:

```rust
use std::fmt;

#[derive(Debug)]
enum AppError {
    NotFound(String),
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
}

// Display = the human-facing message (like Go's Error() string).
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::NotFound(k) => write!(f, "not found: {k}"),
            AppError::Io(e) => write!(f, "io error: {e}"),
            AppError::Parse(e) => write!(f, "parse error: {e}"),
        }
    }
}

// Opting into the std error trait makes it a "real" error.
impl std::error::Error for AppError {}

// These From impls are what make `?` convert automatically (see §3).
impl From<std::io::Error> for AppError {
    fn from(e: std::io::Error) -> Self { AppError::Io(e) }
}
impl From<std::num::ParseIntError> for AppError {
    fn from(e: std::num::ParseIntError) -> Self { AppError::Parse(e) }
}
```

With those `From` impls in place, `?` now converts into `AppError` for free:

```rust
fn load_port(path: &str) -> Result<u16, AppError> {
    let data = std::fs::read_to_string(path)?;          // io::Error -> AppError::Io
    let port = data.trim().parse::<u16>()?;             // ParseIntError -> AppError::Parse
    if port == 0 {
        return Err(AppError::NotFound("port".into()));
    }
    Ok(port)
}
```

```go
// Go — the rough equivalent: a custom error type + Error() method
type AppError struct {
    Kind string
    Err  error
}
func (e *AppError) Error() string { return e.Kind + ": " + e.Err.Error() }
func (e *AppError) Unwrap() error { return e.Err } // enables errors.Is/As
```

The Rust enum buys you something Go's interface doesn't: callers can `match` on
the *variant* exhaustively (`AppError::NotFound(_) => …`), and the compiler
checks they've covered every case. Go's `errors.As`/type-switch gets you partway
there, but nothing forces exhaustiveness. Yes, the boilerplate above is real —
which is exactly why §9's crates exist to generate it.

---

## 8. `Box<dyn Error>` and the `main` trick

When you *don't* need callers to match on specific cases — typically in
application code and `main` — use `Box<dyn std::error::Error>`: a trait object
(Part 5) that means "any type implementing `Error`." Every standard error
converts into it via `?`, so you can freely mix error types with zero `From`
impls of your own:

```rust
fn run() -> Result<(), Box<dyn std::error::Error>> {
    let data = std::fs::read_to_string("config.txt")?; // io::Error
    let port: u16 = data.trim().parse()?;              // ParseIntError
    println!("port = {port}");
    Ok(())
}
```

And `main` itself can return `Result`. If it returns `Err`, the runtime prints
the `Debug` form and exits with a non-zero status — no manual `os.Exit`:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    run()?;
    Ok(())
}
```

```go
// Go — the hand-rolled version of the same thing
func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

`Box<dyn Error>` is the closest Rust gets to Go's universal `error` interface:
heterogeneous, dynamically dispatched, "I just want *an* error." The trade-off
is the same as Go's — you lose static knowledge of *which* error it is, so it's
right for apps and wrong for libraries whose callers need to branch.

---

## 9. The ecosystem: `thiserror` and `anyhow`

The §7 boilerplate (Display, Error, a `From` per variant) is mechanical, so two
crates own this space. The split mirrors the library-vs-app distinction.

**`thiserror`** — for *library* error enums. It derives everything in §7 from
attributes:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("not found: {0}")]
    NotFound(String),

    #[error("io error")]
    Io(#[from] std::io::Error),       // #[from] generates the From impl

    #[error("parse error")]
    Parse(#[from] std::num::ParseIntError),
}
```

That's the *entire* §7 file. `#[error("…")]` writes `Display`; `#[from]`
generates the `From` impl that powers `?`; `Error` is derived. You still get a
precise, matchable enum — you just don't hand-write the plumbing.

**`anyhow`** — for *applications*, where you mostly want to propagate and add
context, not match. `anyhow::Result<T>` is `Result<T, anyhow::Error>`, and
`anyhow::Error` is a souped-up `Box<dyn Error>` with backtraces and a `.context`
method:

```rust
use anyhow::{Context, Result};

fn load_port(path: &str) -> Result<u16> {
    let data = std::fs::read_to_string(path)
        .with_context(|| format!("reading config at {path}"))?;
    let port = data.trim().parse::<u16>()
        .context("config port must be a number")?;
    Ok(port)
}
// An error now reads:
//   config port must be a number
//   Caused by: invalid digit found in string
```

```go
// Go — .context(...) is exactly fmt.Errorf with %w wrapping
data, err := os.ReadFile(path)
if err != nil {
    return 0, fmt.Errorf("reading config at %s: %w", path, err)
}
```

The mapping is tight: `anyhow`'s `.context("…")` ≈ Go's `fmt.Errorf("…: %w",
err)`; both attach a human breadcrumb while preserving the underlying cause. And
`anyhow` has the `errors.Is`/`errors.As` equivalents too — `err.downcast_ref::<
T>()` to recover a concrete type, and `.is::<T>()` to test for one.

Choosing: **`thiserror` when you're a library** and callers need to match
specific errors; **`anyhow` when you're an application** (or a binary's
`main`) and you just want propagation plus context. They compose — a library
defines `thiserror` enums, the app consuming it bubbles them through `anyhow`.

---

## 10. `Result`/`Option` combinators

Sometimes a `match` or `?` is overkill and a method reads better. The common
ones, with what they replace:

```rust
let n: Result<i32, _> = "5".parse();

n.map(|x| x * 2);              // transform the Ok value: Ok(5) -> Ok(10)
"x".parse::<i32>().map_err(AppError::Parse); // transform the Err type
let s: Option<i32> = Some(3);
s.ok_or("was none")?;         // Option -> Result (bridges §4's channel gap)
let p = "bad".parse::<u16>().unwrap_or(8080); // Err -> a default value, no panic
let q = "bad".parse::<u16>().unwrap_or_else(|_| compute_default());
```

```go
// Go — these are the little if-err helpers you write inline
n, err := strconv.Atoi("bad")
if err != nil {
    n = 8080 // unwrap_or(8080)
}
```

`unwrap_or` / `unwrap_or_else` / `unwrap_or_default` are the safe cousins of
`unwrap` — they supply a fallback instead of panicking, so reach for them
whenever "on error, use this default" is the actual intent. `ok_or` is the
bridge you need when `?`-ing an `Option` inside a `Result`-returning function
(§4). Use combinators for short transforms; drop back to `?` or `match` the
moment the logic branches or you'd be chaining more than two or three.

> **Trap:** combinators on `Result` short-circuit, but `map` does **not** run on
> the `Err` case and `map_err` does **not** run on the `Ok` case — they each
> touch only their half. If you find yourself chaining `.map` and `.map_err`
> three deep to express branching logic, you've outgrown combinators; a plain
> `match` will be clearer and the compiler will thank you with better error
> messages.

---

## One-sentence mental model

Errors are values in a `Result<T, E>` just like Go's `(val, err)`, but instead
of hand-writing `if err != nil { return err }` everywhere you write `?`, which
unwraps the `Ok` or early-returns the `Err` — auto-converting it via `From` —
while `panic!`/`unwrap` stay reserved for the unrecoverable bugs you'd reach for
Go's `panic` to express.

---

[← Traits and generics](05-traits-and-generics.md) · [Next: Collections, iterators, closures →](07-collections-iterators-closures.md)
