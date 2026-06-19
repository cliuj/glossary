# A Go developer's guide to Rust

A beginner→intermediate guide to Rust for someone who already programs daily in
**Go**. It doesn't teach programming — it maps Rust onto the Go you already know,
and is honest about where the mapping breaks. Those breaks are usually where
Rust's interesting ideas live: ownership, the borrow checker, traits, and a type
system that does real work.

Go is a deliberate choice of guide rail. The two languages aim at the same broad
target — systems and backend software that's fast and concurrent — but make
opposite bets. Go bets on a garbage collector and a small language; Rust bets on
a compiler that proves memory and thread safety at compile time, with no runtime.
Almost every Rust feature that surprises a Go developer is downstream of that one
bet. Keep it in mind and most of the surprises stop being arbitrary.

**What this guide is not:** an installation tutorial (it assumes a working
`rustc` and `cargo`), a reference manual, or a replacement for the focused
deep-dives that live next to it in this folder.

---

## The five mindset shifts

Everything unfamiliar about Rust traces back to one of these. Keep them in your
pocket; the guide points back to them constantly.

1. **Ownership replaces the garbage collector.** Every value has exactly one
   owner; when the owner goes out of scope, the value is freed — deterministically,
   no GC pause, no `runtime`. Passing a value can *move* it, after which the old
   name is dead. This is the idea the whole language is built around (Part 2).
2. **The borrow checker is a compile-time data-race and use-after-free detector.**
   You can lend out read-only references freely, or exactly one mutable reference,
   but never both at once. In Go you avoid data races by discipline and the `-race`
   detector at runtime; in Rust the compiler rejects them before the program runs.
3. **The type system is the design tool, not paperwork.** Enums are real sum types
   (`Option<T>` replaces `nil`, `Result<T, E>` replaces `(val, err)`), and `match`
   forces you to handle every case. You make illegal states stop compiling rather
   than guarding against them at runtime.
4. **Errors are ordinary values, with one piece of sugar.** There are no
   exceptions for recoverable errors. A function that can fail returns
   `Result<T, E>`, and the `?` operator collapses Go's `if err != nil { return
   ..., err }` into a single character. `panic!` exists, but it's Go's `panic`,
   not your everyday error path.
5. **Abstractions are zero-cost.** Generics, iterators, closures, and traits
   compile down to roughly the code you'd have written by hand — no boxing, no
   virtual dispatch unless you ask for it (`dyn`). You write high-level code and
   pay low-level prices.

One **non**-shift worth stating: Rust is **strictly, eagerly evaluated like Go**.
Expressions run when reached, top to bottom. There's no lazy-evaluation model to
learn (iterators are lazy, but that's a library API, not the language). What
compiles is, to a first approximation, what runs.

---

## Table of contents

| Part | File | What you'll be able to do after |
| ---- | ---- | ------------------------------- |
| 1 | [Reading Rust](01-reading-rust.md) | Parse any line of basic Rust: `fn`, `let`/`let mut`, expression-blocks, `&`/`*`, `match`, `impl`, paths, attributes, and the `!` on macros. |
| 2 | [Ownership and borrowing](02-ownership-and-borrowing.md) | Understand moves, `Copy`, and `&T`/`&mut T`; satisfy the borrow checker; read code full of `&` and know why it's there. |
| 3 | [Lifetimes](03-lifetimes.md) | Read and write `'a` annotations, understand elision, and know when the compiler needs your help and when it doesn't. |
| 4 | [Types and data](04-types-and-data.md) | Model data with structs and enums; replace `nil` with `Option` and error-tuples with `Result`; use `match`, `if let`, and `derive`. |
| 5 | [Traits and generics](05-traits-and-generics.md) | Use traits where you'd use Go interfaces (and where you can't); write generic code with bounds; choose between generics and `dyn`. |
| 6 | [Error handling](06-error-handling.md) | Replace `if err != nil` with `Result` and `?`; build custom error types; know when to `panic!` and when `Box<dyn Error>`/`anyhow` fit. |
| 7 | [Collections, iterators, closures](07-collections-iterators-closures.md) | Use `Vec`/`HashMap`/`String`; replace loops with lazy iterator chains; write closures and read `Fn`/`FnMut`/`FnOnce`. |
| 8 | [Smart pointers and interior mutability](08-smart-pointers-and-interior-mutability.md) | Reach for `Box`/`Rc`/`Arc`/`RefCell` deliberately; understand `Deref` and `Drop` (RAII); recognize and avoid reference cycles. |
| 9 | [Concurrency](09-concurrency.md) | Spawn threads, move data across them, share with `Arc<Mutex<T>>`, and read `Send`/`Sync` as compiler-checked guarantees. |
| 10 | [Async](10-async.md) | Read and write `async`/`.await`, pick a runtime (`tokio`), spawn tasks, and understand how this differs from goroutines. |
| 11 | [Modules, crates, and Cargo](11-modules-crates-and-cargo.md) | Navigate `mod`/`use`/visibility, `Cargo.toml`, workspaces, and tests; map the whole toolchain onto the `go` tool. |

Parts 1–4 are the beginner tier, 5–7 the bridge, 8–11 the intermediate tier.
They build in order, but each part opens with a recap so you can jump back in
cold. Part 2 (ownership) is the one to read slowly — almost everything later
leans on it.

### Sibling deep-dives in this folder

When the guide reaches these topics it covers the essentials and hands off:

- [`../macros.md`](../macros.md) — why `println!` has a `!`, reading
  `macro_rules!`, and what derive/procedural macros actually do. Natural
  follow-up to Part 1 or 5.
- [`../unsafe-and-ffi.md`](../unsafe-and-ffi.md) — what `unsafe` really permits
  (it's narrower than it sounds), raw pointers, and calling C. Natural follow-up
  to Part 8.

---

## Conventions

- Rust has no REPL, so snippets are real code. Output is shown in a comment on
  the line that produces it:

  ```rust
  let doubled: Vec<i32> = [1, 2, 3].iter().map(|x| x * 2).collect();
  println!("{doubled:?}"); // prints: [2, 4, 6]
  ```

  Where a snippet needs surrounding `fn main()` or imports to compile, that's
  shown; otherwise assume it sits inside a function.
- Comparison snippets are labeled `// Go`. Every concept gets the one Go analogy
  that makes it clearest — sometimes that analogy is "Go has no equivalent, and
  here's why."
- Beginner traps are flagged in callouts:

  > **Trap:** like this.

- ⚡ marks a **where-the-Go-analogy-breaks** note — the spots where reaching for
  your Go intuition will mislead you.
- Examples target the **Rust 2021 edition** on stable Rust. Async examples use
  the **tokio** runtime (Part 10 notes the alternatives).

---

## One-sentence mental model

Rust is what you get when "free memory deterministically, forbid data races, and
make illegal states unrepresentable" stops being discipline you apply on top of a
runtime and becomes the language itself — proven by the compiler, with no garbage
collector underneath.
