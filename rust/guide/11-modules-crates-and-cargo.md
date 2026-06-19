# Part 11 — Modules, crates, and Cargo

[← Async](10-async.md) · [Overview](00-overview.md)

This is the "how do real projects fit together and how do I build them" part.
Everything you reach for daily in Go — `go.mod`, `go build`, `go test`,
`gofmt`, package boundaries — has a Rust counterpart, and the mapping is close
enough to be comforting and different enough to trip you. The biggest mental
shift: in Go a directory *is* a package, full stop. In Rust a single crate can
hold an arbitrarily deep tree of modules, and visibility is decided per module
rather than per identifier. Let's line the two worlds up.

---

## 1. Vocabulary, mapped to Go up front

Rust splits "unit of code" into more layers than Go does. Here is the whole
toolchain side by side so the rest of the part has names to hang on.

| Rust | What it is | Closest Go thing |
|------|------------|------------------|
| **crate** | one compilation unit; a single `.rlib` or executable | a Go *package* (a compiled unit) |
| **module** (`mod`) | an in-crate namespace, can nest | roughly a Go package/dir, but lives *inside* a crate |
| **package** | a `Cargo.toml` + its crates | a Go *module* (the `go.mod` root) |
| **`Cargo.toml`** | manifest: name, deps, metadata | `go.mod` |
| **`Cargo.lock`** | pinned exact dep versions | `go.sum` + the resolved `go.mod` graph |
| **cargo** | build/test/dep tool | the `go` tool |
| **crates.io** | the package registry | the module proxy + pkg.go.dev |
| **`use`** | bring a path into scope | `import` |

The trap is that **package** means opposite-ish things. A Go "package" is a
compile unit (≈ Rust crate); a Rust "package" is the `Cargo.toml`-rooted project
(≈ Go module). Keep that swap in mind for the whole part.

> **Trap:** Do not read Rust "package" as Go "package." Rust package ≈ Go
> module; Rust crate ≈ Go package. The words are crossed.

---

## 2. Modules: the tree inside a crate

In Go, code organization *is* the filesystem: a directory is a package, and
that's the only nesting there is — packages don't contain sub-packages in any
language-level sense. Rust gives you a **module tree** that lives inside one
crate, rooted at a node called `crate`.

You make modules three ways.

```rust
// 1. Inline: the whole module body sits right here.
mod math {
    pub fn square(x: i64) -> i64 {
        x * x
    }
}

// 2. As a sibling file: `mod net;` tells the compiler to load `net.rs`
//    (or `net/mod.rs`) and graft its contents in as module `net`.
mod net;

// 3. Nested: modules inside modules, to any depth.
mod http {
    pub mod client {
        pub fn get(url: &str) -> String { /* ... */ }
    }
}
```

```go
// Go
// There is no inline-module equivalent. Organization is files-in-dirs:
//   math/math.go        -> package math
//   net/net.go          -> package net
//   http/client/client.go -> package client
// Each directory is its own compiled unit; nesting is just folders, not a
// language-level tree you address with paths.
```

`mod net;` is the line Go developers stumble on. It is *not* an import. It is
you telling the compiler "this module is part of my crate; go find its source."
A file that is never named by some `mod` statement is simply **not compiled**.
There is no automatic "every `.rs` in the dir is included" rule like Go's.

⚡ *Where the Go analogy breaks: adding a `.rs` file does nothing until a `mod`
declares it. Go compiles every `.go` file in the package directory
automatically; Rust requires you to wire each module in by hand.*

### Paths: `crate::`, `super::`, `self::`

Within the tree you address items by path, much like a filesystem.

```rust
mod http {
    pub mod client {
        pub fn get() {}
    }
    pub mod server {
        pub fn start() {
            // absolute, from the crate root:
            crate::http::client::get();
            // relative, going up one level to `http`:
            super::client::get();
            // relative, within the current module:
            self::helper();
        }
        fn helper() {}
    }
}
```

`crate::` is the root of *this* crate (think the module-path equivalent of an
absolute import inside your own module). `super::` is the parent module — there
is no Go analog because Go packages have no parent. `self::` is the current
module, mostly used for disambiguation.

---

## 3. Visibility: private by default, per module

Go's rule is one line: a `Capitalized` identifier is exported from its package,
a lowercase one is not. It's per-identifier and based purely on spelling.

Rust is **private by default** and you opt out with `pub`. Privacy is scoped to
the *module*, and there are graduated levels:

```rust
mod store {
    pub struct Db { url: String }          // type public, field PRIVATE

    pub fn open() -> Db { /* ... */ }       // anyone can call
    pub(crate) fn migrate() {}              // visible anywhere in THIS crate
    pub(super) fn reset() {}                // visible to the parent module
    fn connect() {}                         // private to `store`
}
```

```go
// Go
package store

type Db struct{ url string } // Db exported, url unexported (lowercase)

func Open() Db  { /* ... */ } // exported
func migrate()  {}            // unexported = package-private; no finer grades
```

Note `Db`'s field `url` is private even though `Db` itself is `pub` — exactly
the struct-field privacy from Part 4 (§ on structs). The type crossing a module
boundary does not drag its fields along. Go has the same field-level rule, just
expressed by capitalization instead of `pub`.

The graduated levels are the real new thing. Go has exactly one notch
(package-private vs exported). Rust gives you `pub(crate)` for "internal API,
not for downstream users" — gold for library crates — plus `pub(super)` and
even `pub(in some::path)` for surgical exposure.

> **Trap:** `pub` on a struct does **not** make its fields public. You must
> mark each field `pub` too, or expose constructors/accessors. Forgetting this
> is the #1 "why can't I read `x.field`" confusion coming from Go.

---

## 4. `use`: bringing paths into scope

Typing `crate::http::client::get()` everywhere is miserable, so `use` pulls a
path into the current scope. This is the closest thing to a Go `import`, but it
operates at the **item** level, not the package level.

```rust
use std::collections::HashMap;          // now `HashMap` names the type
use std::io::Write as IoWrite;          // rename to avoid a clash
use std::fmt::{Debug, Display};         // nested: pull several at once
use crate::http::client;               // bring in a module, call client::get()

fn demo() {
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert("a".into(), 1);
}
```

```go
// Go
// You always import a PACKAGE PATH, then qualify every symbol with the
// package name. There is no symbol-level import and no per-symbol rename
// (only a whole-package alias).
import (
    "fmt"
    iox "io" // alias the whole package, not a single symbol
)
// usage is always fmt.Println, iox.Writer — qualifier required.
```

That's the daily-driver difference: Go forces `pkg.Symbol` qualification and
you import paths; Rust lets you `use` an individual type or function and then
write it bare. Both styles have fans; Rust's is terser, Go's makes origins
obvious at the call site.

⚡ *Where the Go analogy breaks: Rust can import a single function or type and
call it unqualified. Go has no symbol-level import — you always go through the
package name. A bare `parse()` in Rust could come from anywhere a `use`
brought in; in Go it's always `pkg.Parse()`.*

### `pub use`: re-exports and the facade pattern

`pub use` re-exports a path so callers see it at *your* module's level. This is
how crates build a clean public API (a "facade" or "prelude") over a messy
internal tree.

```rust
// In src/lib.rs — flatten internals into a tidy surface:
mod engine;
mod parser;

pub use engine::Runner;        // users write `mycrate::Runner`,
pub use parser::parse;         // not `mycrate::engine::Runner`
```

Go has no direct equivalent; the convention there is to just place the types in
the package you want users to reach, since there's no sub-tree to flatten. The
`prelude` module (`use tokio::prelude::*`) you've seen is exactly this pattern:
a curated bundle of `pub use`s.

---

## 5. Packages and crates: what a project holds

A **package** is one `Cargo.toml`. It contains *at most one* library crate and
*any number* of binary crates, by convention:

```toml
# Cargo.toml
[package]
name = "myapp"
version = "0.1.0"
edition = "2021"

[dependencies]
```

- `src/lib.rs` → the library crate (root module of the lib).
- `src/main.rs` → a binary crate named after the package.
- `src/bin/tool.rs` → an extra binary crate named `tool`.

```go
// Go
// A Go module (go.mod) holds many packages, and `package main` in some
// directory is your executable. Multiple binaries = multiple `main`
// packages in different dirs, all under one module.
//   cmd/myapp/main.go   -> package main (binary)
//   cmd/tool/main.go    -> package main (another binary)
//   internal/...        -> library packages
```

So the lib-plus-many-bins shape maps onto Go's "one module, a `cmd/` dir of
`main` packages, plus library packages." The difference is that Rust's binaries
typically depend on *the package's own library crate* — you put logic in
`lib.rs` and keep `main.rs` thin, just like keeping `cmd/` thin over internal
packages in Go.

---

## 6. `Cargo.toml` and dependencies

This is your `go.mod`. Dependencies are declared with semver ranges, and
`cargo add` edits the file for you (like `go get`).

```bash
cargo add serde --features derive
cargo add tokio --features full
cargo add --dev proptest          # a dev-dependency
```

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }

[dev-dependencies]               # only compiled for tests/examples/benches
proptest = "0.5"
```

`version = "1.0"` is a **caret** range by default: it means `>=1.0.0, <2.0.0`.
Go's modules also follow semver but pin an exact minimum version in `go.mod`
and resolve with Minimal Version Selection; Cargo resolves to the *newest*
compatible version within the range. `Cargo.lock` then records the exact
resolved versions — its job is `go.sum` plus the pinned graph. Commit it for
binaries; for libraries it's typically left out of dependence by downstream.

### Features — the thing Go simply doesn't have

`features` are named, optional compile-time flags a crate exposes, letting you
switch parts of it on or off. `serde`'s `derive` feature pulls in the proc-macro
machinery only if you ask. Enabling a feature can also pull in extra optional
dependencies. **Go has no equivalent concept** — no conditional compilation of a
dependency's surface via flags. The nearest Go cousin is build tags, but those
gate *your* files, not a dependency's feature set.

⚡ *Where the Go analogy breaks: there is no Go feature-flag system. A Rust dep
can compile to wildly different code depending on which features you enable, so
a build that works for you can fail for someone who enabled a different set.*

> **Trap:** Caret ranges mean `cargo` can silently pick a newer patch/minor on
> a fresh checkout *unless* `Cargo.lock` is present and committed. If a build is
> "works on my machine," check whether the lock file is in version control.

---

## 7. The cargo workflow, mapped to `go`

Almost every `go` subcommand has a `cargo` twin.

```bash
cargo build              # like `go build` (artifacts in target/)
cargo build --release    # optimized build, like `go build` with full opt
cargo run                # like `go run .`
cargo run --bin tool     # pick a binary when there are several
cargo test               # like `go test ./...`
cargo check              # type-check ONLY, no codegen — no Go equivalent
cargo fmt                # like gofmt / go fmt
cargo clippy             # linter, like go vet + staticcheck
cargo doc --open         # like go doc, but renders full HTML and opens it
```

Two of these deserve a Go developer's attention.

`cargo check` runs the full borrow-checker and type-checker but skips machine
code generation, so it's dramatically faster than a build. There's no `go`
counterpart because Go compiles so fast the distinction never mattered; in Rust,
`cargo check` in a watch loop is the inner-loop workhorse. Reach for it
constantly.

`cargo clippy` is a much richer linter than `go vet`. It carries hundreds of
lints (idiom nudges, perf suggestions, correctness traps) and is closer in
spirit to staticcheck than to `vet`. Run it in CI; it catches real bugs.

Release vs debug is also more pronounced than in Go: `cargo build` defaults to
an unoptimized **debug** profile (fast compile, slow runtime, overflow checks
on); `--release` flips on optimizations and turns off debug assertions. Always
benchmark with `--release`, or you'll measure the wrong thing.

⚡ *Where the Go analogy breaks: a default `cargo build` is NOT representative of
your program's speed. Go's single build mode is roughly always "optimized";
Rust's default debug build can be an order of magnitude slower. Benchmark only
`--release` artifacts.*

---

## 8. Tests live in the source

Go puts tests in `_test.go` files beside the code. Rust's *unit* tests live
**inside the same file**, in a conventional submodule gated by `#[cfg(test)]`
so it's only compiled when testing.

```rust
pub fn add(a: i64, b: i64) -> i64 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;                 // pull the parent module's items into scope

    #[test]
    fn it_adds() {
        assert_eq!(add(2, 3), 5); // panics with a diff on failure
        assert!(add(0, 0) == 0);
    }
}
```

```go
// Go
// add_test.go, separate file, same package:
func TestItAdds(t *testing.T) {
    if got := add(2, 3); got != 5 {
        t.Fatalf("add(2,3) = %d, want 5", got)
    }
}
```

`cargo test` compiles and runs all of them. Because unit tests sit in-module,
they can reach **private** items — handy, and a real difference from Go where a
`_test.go` in the same package can also see unexported names but lives in its
own file.

Rust has three test flavors:

- **Unit tests** — the `#[cfg(test)] mod tests` shown above, in the source file.
- **Integration tests** — files in a top-level `tests/` directory. Each file is
  compiled as its own crate that depends on your library, so it sees only the
  *public* API. This is your "test it as a consumer would" tier.
- **Doc-tests** — code blocks inside `///` doc comments are compiled and run by
  `cargo test`. Go's testable Examples (`func ExampleFoo`) are the direct
  parallel: runnable documentation that fails the build if it goes stale.

```rust
/// Adds two numbers.
///
/// ```
/// assert_eq!(mycrate::add(2, 2), 4);
/// ```
pub fn add(a: i64, b: i64) -> i64 { a + b }
```

> **Trap:** An integration test in `tests/` can't reach `pub(crate)` or private
> items — it's a separate crate. If a test there won't compile because it can't
> see something, that something isn't part of your public API; move the test
> into a `#[cfg(test)] mod tests` in-source instead.

---

## 9. Workspaces: many crates, one repo

A **workspace** ties multiple packages into one build with a shared
`Cargo.lock` and `target/` dir — the monorepo pattern.

```toml
# Top-level Cargo.toml
[workspace]
resolver = "2"
members = ["app", "core", "cli"]
```

```bash
cargo build --workspace        # build every member
cargo test -p core             # target one member
```

```go
// Go
// go.work stitches multiple modules together for local development:
//   go work init ./app ./core ./cli
// Each member still has its own go.mod; go.work just overlays them.
```

The shapes rhyme but differ. A Go workspace (`go.work`) overlays *separate
modules*, each with its own `go.mod`, mainly so you can develop them together
without `replace` directives. A Cargo workspace is tighter: members share one
lock file and one `target/`, and one member can depend on another by relative
path. It's closer to a single multi-module Go repo than to `go.work` per se.

---

## 10. The corners worth knowing exist

Two more features you'll meet eventually:

- **`build.rs`** — a build script: a Rust file Cargo compiles and runs *before*
  building your crate, for code generation, compiling bundled C, or probing the
  environment. Go's nearest cousin is `go generate`, but `build.rs` runs
  automatically as part of every build rather than on demand.
- **`[profile.*]`** — tune optimization and debug settings per profile
  (`opt-level`, `lto`, `codegen-units`, `strip`). Reach for `[profile.release]`
  with `lto = true` when you want maximum runtime speed at the cost of compile
  time. Go exposes far fewer knobs here by design.

Both are escape hatches — fine to ignore until a real need shows up, then read
the Cargo book section for them.

---

## One-sentence mental model

A **crate** is Go's package (a compile unit), a **package** is Go's module (the
`Cargo.toml` root), `mod` builds a private-by-default *tree* inside one crate
that Go's flat directories don't have, and `cargo` is just the `go` tool with
more knobs — features, profiles, and a real type-only `check`.

---

[← Async](10-async.md) · [Overview](00-overview.md)

Further reading: [macros](../macros.md) · [unsafe and FFI](../unsafe-and-ffi.md).
