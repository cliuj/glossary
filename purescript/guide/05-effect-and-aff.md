# Part 5 — Effect and Aff: side effects, sync and async

[← Functor → Applicative → Monad](04-functor-applicative-monad.md) · [Next: Tooling and projects →](06-tooling-and-projects.md)

Mindset shift #4, delivered. Everything in Parts 1–4 was pure: same inputs,
same output, nothing else happens. Real programs log, read the clock, mutate
DOM, and fetch. PureScript allows all of it without giving up purity — and
after Part 4 you already know the mechanism: **`Effect` and `Aff` are
monads**, and do-notation is how you script them. This is also where the
guide's TS analogies stop being analogies: `Effect` and `Aff` compile to
the exact JS patterns you'd write by hand.

---

## 1. `Effect`: an action as a value

A pure function can't log — logging is an effect. The resolution:

> A value of type `Effect a` is a **description of a synchronous action**
> that, when run, performs side effects and yields an `a`. Building and
> combining descriptions is pure; *running* happens at one place — whatever
> `main` evaluates to is what the runtime executes.

This isn't philosophy; it's literally the compiled representation. An
`Effect a` **is a JS thunk `() => a`**, and you've used this exact pattern
in TS whenever you delayed a side effect by wrapping it in a function:

```typescript
// TypeScript
const greet: () => void = () => console.log("hi");
// Holding `greet` logs nothing. Passing it around logs nothing.
// Only calling it logs.
```

```purescript
import Effect (Effect)
import Effect.Console (log)

greet :: Effect Unit       -- a description of "log hi" — compiles to () => log("hi")
greet = log "hi"

main :: Effect Unit        -- main IS the program: the one thunk that gets called
main = greet
```

`Unit` (value: `unit`) is "no interesting result" — `Effect Unit` is TS's
`() => void`. And because effects are values, the signature now *guarantees*
what a function can do: `Int -> Int` cannot touch the DOM, cannot log,
cannot read the clock. In TS you discover a function's side effects by
reading its body; here the type is a contract the compiler enforces.

> **Trap (strictness, from Part 3):** top-level *pure* bindings evaluate at
> module load, eagerly — `checks = expensiveComputation` runs on import,
> just like a top-level `const` in TS. Only `Effect`/`Aff` values wait to be
> run. If you want "compute when asked," make it a function or an `Effect`.

---

## 2. Writing programs: do-notation over `Effect`

All of Part 4 transfers. `Effect`'s `bind` means "run this action, then feed
its result to the next" — so a do-block reads exactly like the imperative
code it compiles to:

```purescript
import Effect.Random (random)

main :: Effect Unit
main = do
  log "rolling..."
  roll <- random                          -- run it, bind the result (a Number)
  let outcome = if roll > 0.5 then "high" else "low"   -- pure: no action run
  log ("you rolled " <> outcome)
```

```typescript
// TypeScript — and this is roughly the emitted JS, too
function main(): void {
  console.log("rolling...");
  const roll = Math.random();
  const outcome = roll > 0.5 ? "high" : "low";
  console.log(`you rolled ${outcome}`);
}
```

`<-` vs `let`, in their natural habitat: `<-` *runs* an `Effect a` and binds
the `a`; `let` names a pure value. `random` has type `Effect Number` — there
is no `Number` in your hands until the `<-`.

> **Trap:** the first error everyone hits — using an `Effect String` where a
> `String` is wanted. The fix is always: bind first (`s <- getValue`), then
> use `s`. There's no inline unwrap; the bind *is* the unwrap.

### Mutable state, when you need it: `Ref`

Part 1 promised that real mutation lives behind an effect. `Ref` is the
mutable cell — `let` in TS, with the mutation visible in the types:

```purescript
import Effect.Ref as Ref

counter :: Effect Int
counter = do
  ref <- Ref.new 0                -- create:  let c = 0
  Ref.modify_ (_ + 1) ref         -- mutate:  c += 1
  Ref.modify_ (_ + 1) ref
  Ref.read ref                    -- read:    c
```

Every operation is an `Effect`, so a function that touches a `Ref` says so
in its signature. You'll want this rarely in app code (UI frameworks manage
state for you, Parts 8–9), but it's the honest primitive underneath.

---

## 3. Exceptions exist — at the boundary

JS can throw, so PureScript-on-JS can too: `Effect.Exception` has `throw`,
and — more usefully — `try`, which converts a throwing action into the
`Either` you already know how to chain:

```purescript
import Effect.Exception (try, message)

main :: Effect Unit
main = do
  result <- try riskyAction            -- Effect (Either Error a)
  case result of
    Left err -> log ("failed: " <> message err)
    Right a  -> log ("got: " <> show a)
```

The working split, identical in spirit to disciplined TS:

- **Pure code: failure is a return value** — `Maybe`/`Either` in the
  signature ([Part 2 §4](02-types-and-data.md#4-maybe-and-either-absence-and-failure-in-the-type)).
  Pure functions don't throw, full stop.
- **Effectful code: exceptions are in play** (yours, a library's, the
  platform's) — catch with `try` at the boundary where you can respond,
  convert to `Either`, and let the pure core never know exceptions exist.

---

## 4. `Aff`: the better Promise

`Effect` is synchronous. For async work — HTTP, timers, anything
`await`-shaped — PureScript has **`Aff`**, and here the Promise
correspondence is nearly exact, with the differences all in `Aff`'s favor:

| | TS `Promise<A>` | `Aff a` |
| --- | --- | --- |
| starts running | immediately on creation | only when launched — an `Aff` is a *description*, composable and reusable |
| sequencing | `await` / `.then` | `<-` in a do-block (it's `bind`) |
| errors | rejection, `try/catch` | built-in error channel: `attempt`, `throwError` |
| cancellation | manual `AbortController` plumbing | built in — killing an `Aff` runs its cleanup |
| sync code inside | just write it | `liftEffect` (explicit, so the types stay honest) |

```purescript
import Effect.Aff (Aff, launchAff_, delay, attempt)
import Effect.Class (liftEffect)
import Data.Time.Duration (Milliseconds(..))

program :: Aff Unit
program = do
  liftEffect (log "starting")         -- embed a sync Effect into Aff
  delay (Milliseconds 500.0)          -- non-blocking sleep — no callback in sight
  result <- attempt fetchUser         -- attempt :: Aff a -> Aff (Either Error a)
  case result of
    Left err   -> liftEffect (log "fetch failed")
    Right user -> liftEffect (log ("hello " <> user.name))

main :: Effect Unit
main = launchAff_ program             -- launch: the Effect that starts the Aff
```

```typescript
// TypeScript — the same program
async function program(): Promise<void> {
  console.log("starting");
  await sleep(500);
  try {
    const user = await fetchUser();
    console.log(`hello ${user.name}`);
  } catch {
    console.log("fetch failed");
  }
}
```

Read `Aff` do-blocks as `async` function bodies: each `<-` is an `await`,
except nothing runs until `launchAff_` (the `Aff`/`Promise` eager-vs-lazy
difference — it's why an `Aff` can be retried, raced, or composed before
ever starting). The two glue functions to internalize:

- **`liftEffect :: Effect a -> Aff a`** — sync actions go *into* `Aff`
  freely. (If the compiler couldn't match `Effect` with `Aff`, this is what
  it wants.)
- **`launchAff_ :: Aff a -> Effect Unit`** — an async program is *started*
  from the sync world, typically once, in `main`. Part 7 adds the third
  glue: `Promise` ↔ `Aff`, for meeting JS libraries.

Concurrency note for later: `Aff` has `forkAff`, `joinFiber`, and
`parallel`/`sequential` — `Promise.all` and friends, but cancelable and
composable. File it under "exists when you need it."

---

## 5. The boundary is the architecture

The deeper consequence of effects-in-types: the pure/impure split becomes a
design line the compiler patrols. You already aim for this in TS — thin
handlers, logic in plain testable functions, effects at the edges. Here it's
load-bearing:

```purescript
-- THE CORE: pure decisions — trivially testable, no mocks
validateOrder :: Cart -> Either OrderError Order
priceOrder    :: Catalog -> Order -> PricedOrder

-- THE SHELL: effectful rim that ferries data in and out
submitOrder :: Cart -> Aff Unit
submitOrder cart = do
  catalog <- fetchCatalog                          -- effect
  case priceOrder catalog <$> validateOrder cart of -- pure sandwich
    Left err     -> liftEffect (log (show err))
    Right priced -> postOrder priced               -- effect
```

Working heuristic: when a function wants `Effect`/`Aff` just to *obtain* a
value, flip it — do the effect in the caller, pass the value in
(`priceOrder :: Catalog -> ...`, not `priceOrder :: Aff ...`). Keep the rim
thin; UI frameworks (Parts 8–9) are structured around exactly this split.

---

## One-sentence mental model

`Effect a` is a typed `() => a` thunk and `Aff a` a lazy, cancelable
`Promise<a>` — both are *descriptions* that pure code builds and `main`
runs, do-notation chains them like async/await (`liftEffect` embeds sync in
async, `launchAff_` starts the async world), exceptions get `try`-ed into
`Either` at the rim, and the pure core never knows any of it happened.

---

[← Functor → Applicative → Monad](04-functor-applicative-monad.md) · [Next: Tooling and projects →](06-tooling-and-projects.md)
