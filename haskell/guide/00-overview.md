# A working developer's guide to Haskell

A beginner→intermediate guide to Haskell for someone who already programs daily
in **TypeScript, Python, or Go**. It doesn't teach programming — it maps Haskell
onto what you already know, and is honest about where the mapping breaks (those
spots are usually where the interesting ideas live).

**What this guide is not:** a tooling/setup tutorial (it assumes a working GHC),
a category theory text, or a replacement for the focused deep-dives that live
next to it in this folder.

---

## The four mindset shifts

Everything unfamiliar about Haskell traces back to one of these. Keep them in
your pocket; the guide will point back to them constantly.

1. **Everything is an expression.** There are no statements. `if`, `case`,
   even a whole function body — each one *is a value*. Think "the entire
   language works like a ternary expression," not "where's my early return."
2. **Immutability is the default and only mode.** A "variable" is a name for a
   value, full stop — like if `const` were the only keyword and objects were
   frozen. Loops become recursion; updates become new values.
3. **Types are the design tool, not the paperwork.** In TS you often write
   code, then types. In Haskell you model the domain in types first, and
   illegal states stop compiling. Sum types do most of the heavy lifting.
4. **Effects are tracked in the types.** A function that does I/O *says so in
   its signature* (`IO`). The compiler enforces the "keep side effects at the
   edges" discipline you already apply by convention in Go and TS.

---

## Table of contents

| Part | File | What you'll be able to do after |
| ---- | ---- | ------------------------------- |
| 1 | [Reading Haskell](01-reading-haskell.md) | Parse any line of basic Haskell: signatures, currying, `$`, `.`, guards, `where`, pattern matching, the layout rule. |
| 2 | [Types and data](02-types-and-data.md) | Model data with records and sum types; use `Maybe`/`Either` instead of `null` and `(val, err)`; read `deriving` and basic typeclasses. |
| 3 | [Functions, lists, laziness](03-functions-lists-laziness.md) | Replace loops with `map`/`filter`/folds and recursion; understand lazy evaluation and the `String`/`Text` split. |
| 4 | [Functor → Applicative → Monad](04-functor-applicative-monad.md) | Read and write do-notation; use `Maybe`/`Either`/list as monads; stop being scared of the M-word. |
| 5 | [IO and effects](05-io-and-effects.md) | Write real programs: `main`, I/O actions, the pure/impure boundary, error handling in practice. |
| 6 | [Structuring programs](06-structuring-programs.md) | Navigate modules, imports, and project layout; recognize Monoid idioms and lens operators in the wild. |
| 7 | [Transformers and the ReaderT pattern](07-transformers-and-readert.md) | Read real application code: monad transformer stacks, `mtl` classes, and the `ReaderT Env IO` architecture. |

Parts 1–3 are the beginner tier, 4–5 the bridge, 6–7 the intermediate tier.
They build in order, but each part opens with a recap so you can jump back in
cold.

### Sibling deep-dives in this folder

When the guide reaches these topics it covers the essentials and hands off:

- [`../typeclasses-newtypes-deriving-via.md`](../typeclasses-newtypes-deriving-via.md)
  — typeclasses vs interfaces in depth, `newtype` mechanics, `deriving via`.
  Natural follow-up to Part 2.
- [`../phantom-types.md`](../phantom-types.md) — type parameters with no
  runtime data: tagged ids, typestate. Natural follow-up to Part 2 or 6.

---

## Conventions

- REPL snippets use the `λ>` prompt; lines without it are file code:

  ```haskell
  λ> map (*2) [1, 2, 3]
  [2,4,6]
  ```

- Comparison snippets are labeled with a comment in their own language —
  `// TypeScript`, `# Python`, `// Go`. Each concept gets the *one* language
  that makes the clearest analogy, not all three.
- Beginner traps are flagged in callouts:

  > **Trap:** like this.

- Examples are small and GHCi-runnable wherever possible. When a snippet needs
  a language extension or import, it's stated at the top of the snippet.

---

## One-sentence mental model

Haskell is what you get when "const everything, isolate side effects, make
illegal states unrepresentable" — advice you already follow by discipline —
becomes the language itself, enforced by the compiler.
