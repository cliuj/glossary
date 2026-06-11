# Part 7 — FFI: using JS/TS libraries (and being used by them)

[← Tooling and projects](06-tooling-and-projects.md) · [Next: Halogen →](08-halogen.md)

The part a real frontend project can't skip. PureScript's ecosystem is good
but small; npm's is infinite. The FFI (foreign function interface) is the
bridge — and because PureScript compiles to ordinary ES modules
([Part 6 §3](06-tooling-and-projects.md#3-project-anatomy)), the bridge is
short: no marshalling layer, no bindings generator, just JS values meeting
typed names. Every snippet in this part was compiled and executed during
writing; the recipes are known-good against PureScript 0.15 / spago 1.x.

**The mental model:** writing FFI is writing `.d.ts` declarations — you
assert types for untyped code and the compiler trusts you. It is the one
place in the language where you can lie, so the craft is (a) small
surfaces, (b) honest types, (c) conversions at the boundary. Lie in an FFI
signature and you get TS-grade runtime surprises in code the compiler
believes is safe — the whole game is keeping the lying zone tiny.

---

## 1. The mechanics: a module and its shadow

A module `FFI/Basics.purs` may have a companion **`FFI/Basics.js`** — same
name, same folder, an ordinary ES module. `foreign import` declares the
type; the JS file provides the value:

```purescript
-- src/FFI/Basics.purs
module FFI.Basics where

foreign import repeatString :: Int -> String -> String
```

```javascript
// src/FFI/Basics.js — note: CURRIED, one arrow per PureScript arrow
export const repeatString = (n) => (s) => s.repeat(n);
```

```purescript
> repeatString 3 "ab"
"ababab"
```

The convention to internalize: **multi-argument PureScript functions are
curried, so the JS implementation must be too** — `Int -> String -> String`
is `(n) => (s) => ...`, exactly the arrow-chain from
[Part 1 §2](01-reading-purescript.md#2-type-signatures-arrows-and-forall).
(§3 shows the uncurried escape hatch you'll actually use for real
libraries.)

Two boundary rules that make life easy:

- **Records are JS objects, verbatim.** `{ base :: Int, bonus :: Int }`
  arrives in JS as `{ base, bonus }` — no conversion in either direction.
  Records are your workhorse for options objects and structured returns.
- **Effects are thunks** ([Part 5 §1](05-effect-and-aff.md#1-effect-an-action-as-a-value)
  was literal): `Effect a` on the PureScript side means `() => a` on the JS
  side:

```purescript
foreign import nowMillis :: Effect Number
```

```javascript
export const nowMillis = () => Date.now();   // the thunk IS the Effect
```

> **Trap:** forget the thunk — `export const nowMillis = Date.now()` — and
> it type-checks, compiles, and snapshots the time *once at module load*,
> forever. Wrong-thunking is the classic FFI bug precisely because the
> compiler can't see JS; when an "effect" returns a suspiciously constant
> value, look here first.

---

## 2. `Nullable`: where `null` re-enters

PureScript has no `null`, JS is made of it. `Data.Nullable` is the typed
airlock:

```purescript
import Data.Nullable (Nullable, toMaybe)
import Effect.Uncurried (EffectFn1, runEffectFn1)

foreign import getEnvImpl :: EffectFn1 String (Nullable String)

getEnv :: String -> Effect (Maybe String)        -- the PUBLIC api: Maybe, not Nullable
getEnv name = toMaybe <$> runEffectFn1 getEnvImpl name
```

```javascript
export const getEnvImpl = (name) => process.env[name] ?? null;
```

`Nullable a` is "an `a` or `null`, JS-style" — it exists *only* to describe
the boundary truthfully. Convert to `Maybe` immediately (`toMaybe`), and
back with `toNullable` when calling in. The pattern above is the standard
FFI shape: a private `...Impl` foreign import with boundary types, and a
public wrapper exposing proper PureScript types. Callers never see the JS
flavor.

---

## 3. Uncurried: meeting JS functions as they are

Real JS functions take their arguments in one go, and wrapping each one in
curry-adapters by hand gets old. `Data.Function.Uncurried` (pure) and
`Effect.Uncurried` (effectful) provide types that say "a *JS-arity*
function":

```purescript
import Data.Function.Uncurried (Fn2, runFn2)

foreign import joinWithImpl :: Fn2 String (Array String) String
--                             ^ a JS function of exactly 2 arguments

joined = runFn2 joinWithImpl ", " ["a", "b"]     -- runFn2 calls it
```

```javascript
export const joinWithImpl = (sep, xs) => xs.join(sep);   // natural JS, no curry
```

`Fn2`–`Fn10` and `EffectFn1`–`EffectFn10` (you met `EffectFn1` in §2) cover
both directions: calling JS, and handing callbacks *to* JS (`mkFn2`,
`mkEffectFn1` wrap a PureScript function into JS-arity form — that's how
event handlers cross). Rule of thumb: foreign imports of real-world JS are
nearly always `Fn`/`EffectFn` + a typed public wrapper.

---

## 4. Wrapping an npm package (worked, verified)

The full pattern on two real packages — `slugify` (pure, options object)
and `nanoid` (effectful):

```bash
npm install slugify nanoid
```

```purescript
-- src/FFI/Npm.purs
module FFI.Npm where

import Data.Function.Uncurried (Fn2, runFn2)
import Effect (Effect)

type SlugOptions = { lower :: Boolean, strict :: Boolean }

foreign import slugifyImpl :: Fn2 String SlugOptions String

slugify :: SlugOptions -> String -> String       -- public: options first,
slugify opts s = runFn2 slugifyImpl s opts       -- pipeline-friendly

foreign import nanoid :: Effect String           -- impure (random) → Effect. Honest types!
```

```javascript
// src/FFI/Npm.js
import slugifyJs from "slugify";
import { nanoid as nanoidJs } from "nanoid";

export const slugifyImpl = (s, opts) => slugifyJs(s, opts);
export const nanoid = () => nanoidJs();
```

```purescript
> slugify { lower: true, strict: true } "Hello FFI World!"
"hello-ffi-world"
> id <- nanoid          -- a fresh id each run — because Effect
```

Note the two judgment calls, because they're the actual skill: `nanoid`
returns a *different string each call*, so its honest type is
`Effect String`, not `String` (the compiler can't make you do this — the
boundary is where *you* uphold purity). And the options record nails down
the two fields you use rather than slugify's full option soup — FFI types
should describe *your usage*, not the library's whole surface. Widen later
when you need to; you can't un-promise.

---

## 5. Promises ↔ `Aff`: the async bridge

The `aff-promise` package (`Control.Promise`) converts in both directions,
which makes the entire async-JS world available as `Aff` —
[Part 5 §4](05-effect-and-aff.md#4-aff-the-better-promise)'s table comes
alive:

```purescript
import Control.Promise (Promise, toAffE)

foreign import fetchTextImpl :: String -> Effect (Promise String)
--                              calling it STARTS the request → Effect of a Promise

fetchText :: String -> Aff String                -- the public face: a proper Aff
fetchText url = toAffE (fetchTextImpl url)
```

```javascript
export const fetchTextImpl = (url) => () =>     -- curried arg, then the Effect-thunk
  fetch(url).then((r) => r.text());
```

`toAffE :: Effect (Promise a) -> Aff a` is the one-liner that ends most FFI
async stories: thunk-wrap the promise-creating call (creating a Promise
*runs* it — that eagerness is exactly why the type is `Effect (Promise a)`
and not `Promise a`), convert, and the caller gets a real `Aff` —
composable, cancelable from the PureScript side, `attempt`-able, with the
rejected-Promise path arriving on `Aff`'s error channel. The reverse
direction (`fromAff`) is in §7.

---

## 6. Calling your TypeScript from PureScript

FFI companion files must be `.js`, but nothing stops them being one-line
re-exports from *compiled TS*. Two verified setups:

**Quick (relative path).** Compile your TS somewhere stable and import it
relative to where FFI files land — which is always two levels deep, at
`output/<Module>/foreign.js`:

```bash
npx esbuild ts/helper.ts --outdir=ts-out --format=esm
```

```javascript
// src/FFI/Score.js — "../../" climbs out of output/FFI.Score/
export { computeScore } from "../../ts-out/helper.js";
```

**Robust (local npm package) — recommended once TS code is substantial.**
Give the TS its own folder with a `package.json`, install it by path, and
import by name — both node and esbuild resolve it with zero path fragility:

```jsonc
// ts-lib/package.json
{ "name": "myapp-ts", "version": "1.0.0", "type": "module", "main": "dist/helper.js" }
```

```bash
npx esbuild ts-lib/helper.ts --outdir=ts-lib/dist --format=esm
npm install ./ts-lib
```

```javascript
// src/FFI/Score.js
export { computeScore } from "myapp-ts";
```

```purescript
-- src/FFI/Score.purs — typing the TS function; records cross as-is
foreign import computeScore :: { base :: Int, bonus :: Int } -> Int
```

Either way the shape is the same: **TS owns the logic, the FFI `.js` file
is a re-export, the `.purs` file is the type you swear to.** Your TS
function takes a single object argument? Then it's already PureScript-arity
(one argument) and needs no `Fn` wrapper — one of several reasons
object-bag APIs age well on both sides of this bridge.

---

## 7. Calling PureScript from TypeScript

The other direction needs no FFI at all — compiled modules are importable
ES modules (verified against `output/` directly; after `spago bundle`, the
same API can be bundled in). Three calling conventions to know, all visible
in [Part 6 §3](06-tooling-and-projects.md#3-project-anatomy)'s emitted JS:

```purescript
-- src/Api.purs — a deliberate TS-facing surface
module Api (add, greet, delayedGreet) where

import Control.Promise (Promise, fromAff)

add :: Int -> Int -> Int
add x y = x + y

greet :: String -> String
greet name = "Hello, " <> name <> "!"

delayedGreet :: String -> Effect (Promise String)   -- Aff exposed as Promise
delayedGreet name = fromAff do
  delay (Milliseconds 20.0)
  pure (greet name)
```

```typescript
// consumer.ts
import { add, greet, delayedGreet } from "./output/Api/index.js";

const sum = add(3)(4);                       // curried: one paren-pair per argument
const hello = greet("TS");                   // 1-arg functions look normal
const msg = await delayedGreet("async")();   // Effect = thunk: note the trailing ()
```

The conventions: **curried calls** (`add(3)(4)`), **`Effect` values are
thunks you invoke** (the trailing `()`), and **`Aff` crosses as `Promise`
via `fromAff`**. For a polished boundary, write a small `.d.ts` by hand for
just the exposed module (`add(x: number): (y: number) => number`, etc.) —
and design that module to export only TS-friendly shapes: records,
primitives, arrays, Promises. Keep `Maybe`, `Either`, and other PureScript
data types on your side of the wall (convert with `toNullable` / a
`{ ok, value, error }` record), because their constructor representation is
an implementation detail TS shouldn't couple to.

---

## 8. The safety rules

Collected from above, the discipline that keeps the FFI a feature instead
of a hole:

1. **Foreign modules are thin leaves.** A `foreign import` + companion file
   per integration, a typed public wrapper in front, no logic in the `.js`
   beyond adaptation. All the lying happens in files you can list.
2. **Honest types at the boundary:** effects are `Effect`/`Aff` even when
   inconvenient (§4's `nanoid`), nullability is `Nullable` converted
   immediately (§2), JS-arity is `Fn`/`EffectFn` (§3), thunks wrap anything
   eager (§1's trap, §5's promises).
3. **Don't trust incoming *shapes*, either.** A signature like
   `fetchUser :: Aff User` claims the server returns a `User` — the FFI
   trusts that claim as blindly as `as User` would in TS. For data from the
   network or other untyped sources, return `Json` and decode it — the
   `argonaut`/`codec-argonaut` libraries are PureScript's zod, producing
   `Either DecodeError User` and pushing malformed data into the error
   channel where Part 4 machinery handles it.
4. **Convert at the boundary, not downstream.** PureScript types in
   PureScript-land, JS types in JS-land, and translation only in the
   wrappers. If a `Nullable` or a curried JS call shows up three modules
   deep, the boundary leaked.

---

## One-sentence mental model

FFI is hand-written `.d.ts` with consequences: a `.purs` declaration types
what a same-named `.js` ES module exports — curried functions, records as
plain objects, `Effect` as thunk, `Promise` as `Aff` via `toAffE`/`fromAff`,
`null` as `Nullable` — so wrap each npm or TS dependency in one thin,
honestly-typed leaf module, decode untrusted shapes like you'd zod them, and
both languages get to keep their worldview.

---

[← Tooling and projects](06-tooling-and-projects.md) · [Next: Halogen →](08-halogen.md)
