# Part 6 — Structuring programs

[← IO and effects](05-io-and-effects.md) · [Next: Transformers and the ReaderT pattern →](07-transformers-and-readert.md)

The language tour is done; this part is about *codebases*. How files become
modules, what the project skeleton looks like, how real code copes with
records at scale, and the handful of idioms that appear in every Haskell
project you'll read. Less narrative, more field guide.

---

## 1. Modules and imports

One file = one module, named after its path: `src/Data/Validation.hs` holds
`module Data.Validation`. The header's export list is the public API:

```haskell
module Order
  ( Order          -- export the TYPE only — constructors stay private
  , OrderId (..)   -- the (..) exports the type AND its constructors
  , mkOrder        -- the sanctioned way to build an Order
  , total
  ) where
```

Omit the list and everything is public (fine for app code, sloppy for
libraries). Note there's no per-definition `export` keyword and no Go-style
capitalization rule — visibility lives in one place, the header.

**The export list is an encapsulation tool**, and its killer use is the
**smart constructor** pattern: export the type but *not* its constructor, so
the only way in is through your validating function —

```haskell
mkOrder :: [Item] -> Either OrderError Order   -- the public, validating door
```

Outside the module, pattern matching on `Order`'s internals is impossible
and invalid `Order`s cannot exist. This is "private constructor + factory
method," with the type system as the access modifier. (It's also the
discipline [the phantom-types notes](../phantom-types.md) lean on to keep
tag relabeling honest.)

### Import styles

```haskell
import Data.List                          -- everything, unqualified (sort, foldl', ...)
import Data.List (sort, foldl')           -- only these names
import qualified Data.Map as Map          -- everything, behind a prefix: Map.lookup
import Data.Map (Map)                     -- the common COMBO:
import qualified Data.Map as Map          --   type unqualified, functions qualified
```

```typescript
// TypeScript equivalents
import { sort, foldl } from "data-list";
import * as Map from "data-map";
```

The combo in the last pair is the idiom to internalize (you saw it with
`Data.Text` in Part 3): signatures get to say `Map String Int` while calls
say `Map.lookup` — necessary because container modules deliberately reuse
Prelude names (`lookup`, `filter`, `map`), and qualification is how they
coexist. Seeing `import qualified Data.Map as M`, `as T`, `as BS` at the top
of a file tells you its vocabulary.

---

## 2. Anatomy of a project

What you'll find when you clone a real repo (reading-level — build commands
are out of scope):

```
myapp/
├── myapp.cabal          -- the manifest: package.json / go.mod
├── src/                 -- the library: nearly all the code
│   └── MyApp/...
├── app/Main.hs          -- thin executable: main, arg parsing, calls the library
└── test/Spec.hs         -- the test suite
```

The `.cabal` file declares components and — the part you'll actually consult
— their dependencies and defaults:

```cabal
library
  hs-source-dirs:     src
  exposed-modules:    MyApp.Order, MyApp.Db
  build-depends:      base, text, containers, aeson
  default-extensions: OverloadedStrings, LambdaCase
```

Two things worth noticing as a reader. First, the split: even applications
put ~all code in `library` with a few-line executable on top — it's the
functional-core/imperative-shell shape from Part 5, at project scale (and
it's what makes the code testable). Second, **`default-extensions`**: those
language pragmas from Part 3 can be enabled project-wide here — so if a file
uses `OverloadedStrings` with no pragma in sight, check the `.cabal` file.
(You may also see `stack.yaml` alongside — Stack is an alternative build
tool reading the same package format.)

---

## 3. Records at scale

Part 2 flagged the wart: field accessors are top-level functions, so two
records with a `name` field clash. How real codebases cope, in the order
you'll encounter them:

**Prefixed fields** — the zero-magic convention, very common:

```haskell
data User  = User  { userName :: Text, userEmail :: Text }
data Order = Order { orderId  :: Int,  orderUserName :: Text }
```

**`OverloadedRecordDot`** (GHC 9.2+) — TS-style dot access, the modern cure:

```haskell
{-# LANGUAGE OverloadedRecordDot #-}
fullLabel u o = u.name <> " / " <> show o.id     -- fields may share names freely
```

**Lenses** — the powerful one you mainly need to *recognize*. A lens is a
first-class getter/setter for a field, and libraries (`lens`, `microlens`)
provide operators for using them. The Rosetta stone:

```haskell
user ^. address . city                  -- GET:    user.address.city
user &  address . city .~ "Oslo"        -- SET:    {...user, address: {...user.address, city: "Oslo"}}
user &  age %~ (+ 1)                    -- UPDATE: apply a function to the field
```

Decoder: `^.` gets, `.~` sets (returning a new record), `%~` modifies with a
function, `&` is reverse application (`x & f = f x` — it pipes the record
into the operations, reading left-to-right), and composing lenses with `.`
reaches into *nested* records. That last bit is the actual selling point:
nested immutable update — the spread-operator pyramid in the SET line's
comment — collapses to one chain. Convention: lens names are the field
prefixed or underscored (`_name`), generated by a `makeLenses` declaration.
Reading fluency is all you need to start; writing your own lenses can wait.

**Strict fields** — the other record habit you'll see, paying off Part 3 §5
(force what you accumulate): a `!` before a field's type makes storing a
value force it, so long-lived records don't quietly pile up thunks. Many
codebases mark every field strict by default:

```haskell
data Stats = Stats { count :: !Int, total :: !Double }
```

---

## 4. Field guide to everyday idioms

The recurring vocabulary of real code, by recognition:

**`Data.Map`** — *the* dictionary type (from `containers`; immutable,
ordered keys). The API is `Maybe`-flavored where TS/Python would
null/throw:

```haskell
import qualified Data.Map as Map

inventory = Map.fromList [("apples", 12), ("pears", 7)]
Map.lookup "apples" inventory       -- Just 12 — total: returns Maybe, never throws
Map.insert "plums" 3 inventory      -- a NEW map (persistent, like Immutable.js)
Map.insertWith (+) "apples" 1 inv   -- upsert-with-combine: the counter idiom
```

**`<>` and `mempty` (Semigroup/Monoid)** — `<>` is "the natural way to
combine two of these," defined per type by a typeclass: concatenation for
lists/`Text`, union for `Map`/`Set`, pointwise for tuples and functions.
`mempty` is the per-type empty (`""`, `[]`, empty map). You've been using
`<>` on strings since Part 1; the abstraction is why generic code can say
"combine all results" (`mconcat`, `foldMap`) without caring what the results
are. When you see a signature with `Monoid m =>`, read: "anything
combinable."

**The `Maybe`/`Either` toolkit** — promised in Part 2; the four to know cold
(all in the Prelude or `Data.Maybe`):

```haskell
fromMaybe :: a -> Maybe a -> a                         -- unwrap with a default: m ?? fallback
maybe     :: b -> (a -> b) -> Maybe a -> b             -- fold: default + function in one call
either    :: (e -> c) -> (a -> c) -> Either e a -> c   -- fold both branches
mapMaybe  :: (a -> Maybe b) -> [a] -> [b]              -- map + keep the hits: flatMap-filter
```

`maybe 0 length mStr` or `either show render result` replace a whole `case`
— once these read naturally, most "noisy" Haskell stops being noisy.

**`LambdaCase`** — cosmetic but ubiquitous: `\case` is a lambda that
immediately pattern-matches its argument:

```haskell
{-# LANGUAGE LambdaCase #-}
describe = \case          -- instead of:  describe x = case x of ...
  0 -> "zero"
  _ -> "many"
```

---

## One-sentence mental model

A Haskell codebase is modules with curated export lists (smart constructors
make invalid values unconstructable), a thin `app/` over a thick `src/`
library described by the `.cabal` file — and its daily dialect is qualified
imports (`Map.lookup`), record access via prefixes, dot-syntax, or lenses
(`^.` gets, `.~` sets), `<>`/`mempty` for combining, and the
`fromMaybe`/`maybe`/`either` toolkit instead of case-cascades.

---

[← IO and effects](05-io-and-effects.md) · [Next: Transformers and the ReaderT pattern →](07-transformers-and-readert.md)
