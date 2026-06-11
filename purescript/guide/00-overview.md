# A TypeScript developer's guide to PureScript

A beginner→intermediate guide to PureScript for someone who programs daily in
**TypeScript** and wants to build frontend UIs with stronger guarantees. It
maps PureScript onto the TS you already know — and since PureScript compiles
to plain JavaScript and lives inside the npm ecosystem, the mapping runs
deeper than analogy: your code ends up as readable JS modules sitting next to
your `node_modules`.

**What this guide is:** the language from the ground up (Parts 1–5), the
toolchain (Part 6), **how to use JS/TS libraries from PureScript and vice
versa** (Part 7 — the part you'll keep coming back to), and the same small UI
built in both Halogen and react-basic-hooks so you can pick a framework
(Parts 8–9).

**Relation to the Haskell guide:** this guide stands alone — but PureScript
is Haskell's closest living relative, so if you've read
[`../../haskell/guide/`](../../haskell/guide/00-overview.md), Parts 1–4 will
read as "Haskell with friendlier defaults" and you can skim for the
⚡ **difference callouts**. If you haven't, ignore them; nothing depends on
Haskell.

---

## The four mindset shifts

1. **Everything is an expression.** No statements — `if`, `case`, a whole
   function body each *are values*. Think "the entire language works like a
   ternary," not "where's my early return."
2. **Immutability is the only mode.** A "variable" is a name for a value, as
   if `const` were the sole keyword and every object deeply frozen. Loops
   become `map`/`filter`/folds; updates produce new values.
3. **Types are the design tool.** You model the domain in types first and
   illegal states stop compiling. The compiler has no `any`, no `as`, no `!`
   — there is no escape hatch to silence it, and (outside Part 7's FFI
   boundary) no runtime type errors to have.
4. **Effects are tracked in the types.** A function that touches the DOM,
   logs, or fetches says so in its signature (`Effect`, `Aff`). The "keep
   side effects at the edges" discipline you apply by convention becomes
   compiler-enforced.

One **non**-shift worth celebrating up front: PureScript is **strictly
evaluated, like TypeScript** — expressions run when encountered, top to
bottom, no lazy-evaluation model to learn. What compiles is roughly the JS
you'd have written.

---

## Table of contents

| Part | File | What you'll be able to do after |
| ---- | ---- | ------------------------------- |
| 1 | [Reading PureScript](01-reading-purescript.md) | Parse any line: signatures, currying, `$`/`#`, `<<<`, guards, pattern matching, the layout rule. |
| 2 | [Types and data](02-types-and-data.md) | Model with records (they're TS objects, but better-typed) and sum types; use `Maybe`/`Either`; read type classes and `derive`. |
| 3 | [Functions, arrays, purity](03-functions-arrays-purity.md) | Replace loops with `map`/`filter`/folds; know the `Array`/`List`/`String` landscape and the recursion rules of a strict language. |
| 4 | [Functor → Applicative → Monad](04-functor-applicative-monad.md) | Read and write do-notation; chain `Maybe`/`Either`/`Array`; see why async/await was monads all along. |
| 5 | [Effect and Aff](05-effect-and-aff.md) | Write real programs: `main`, synchronous effects, `Ref` mutable state, and `Aff` — the better `Promise`. |
| 6 | [Tooling and projects](06-tooling-and-projects.md) | Navigate `spago.yaml`, the registry, the build pipeline, and the dev loop — mapped onto `npm`/`tsc`/`vite`. |
| 7 | [FFI: using JS/TS libraries](07-ffi-typescript.md) | Wrap any npm package in typed PureScript: foreign imports, `EffectFn`, `Nullable`, `Promise`↔`Aff`, and calling PureScript from TS. |
| 8 | [Halogen](08-halogen.md) | Build a component the all-PureScript way: state, actions, effects, the type-safe HTML DSL. |
| 9 | [react-basic-hooks](09-react-basic-hooks.md) | Build the same component on real React with hooks — then choose between the two frameworks with a clear head. |

Parts 1–5 are the language tier, 6–7 the integration tier, 8–9 the UI tier.
Each part closes with a one-sentence mental model; they build in order.

---

## Conventions

- REPL snippets (the REPL is `spago repl`, Part 6) use the `>` prompt; lines
  without it are file code:

  ```purescript
  > map (_ * 2) [1, 2, 3]
  [2,4,6]
  ```

- TypeScript comparison snippets are labeled `// TypeScript`.
- Beginner traps are flagged in callouts:

  > **Trap:** like this.

- ⚡ marks a **for-Haskell-readers** difference callout — skippable if
  Haskell isn't in your toolbox.
- Examples assume PureScript 0.15+ and spago 1.x (the `spago.yaml` era), and
  state their imports unless they're from the Prelude.

---

## One-sentence mental model

PureScript is "what if TypeScript's type system kept every promise it makes"
— a strict, JS-targeting language where `const` is the only mode, effects
appear in signatures, there is no `any`, and the escape hatch to the JS
ecosystem is one explicit, typed boundary you control.
