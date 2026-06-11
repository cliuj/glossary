# Part 2 — Types and data

[← Reading PureScript](01-reading-purescript.md) · [Next: Functions, arrays, purity →](03-functions-arrays-purity.md)

Mindset shift #3: you design in types first. This part covers PureScript's
two modeling tools — **records** (which will feel like TS object types,
because they nearly are) and **sum types** (which TS approximates with
discriminated unions and Go users envy) — plus `Maybe`/`Either`, `newtype`,
and enough type classes to read real code.

---

## 1. The primitives: where PureScript meets JS

PureScript compiles to JS, and its primitive types are deliberately JS's own:

| PureScript | Compiles to | Notes |
| ---------- | ----------- | ----- |
| `String` | JS string | a primitive, like TS — with full `Data.String` API |
| `Boolean` | JS boolean | `true` / `false`, lowercase |
| `Number` | JS number | a double, exactly TS's `number`; literals need a dot: `2.0` |
| `Int` | JS number, kept integral | a *compiler-enforced* integer JS never had; literals: `2` |
| `Array a` | JS array | *the* collection type ([Part 3](03-functions-arrays-purity.md)) |
| records | JS object | §2 — the headline |

> **Trap:** `Int` and `Number` are distinct types that never mix silently —
> `1 + 2.5` won't compile, because `+`'s two sides must be the same type.
> Convert explicitly: `toNumber :: Int -> Number` (from `Data.Int`). After a
> career of `0.1 + 0.2` surprises in floats-only JS, having real integers is
> a quiet luxury: array indices, counts, and money-in-cents stay exact by
> construction. (One bound to know: `Int` is 32-bit and wraps on overflow —
> for values past ±2.1 billion use `Number` or a big-integer library.)

---

## 2. Records: TS object types, structurally typed

PureScript records *are* JS objects with a type — and the typing is
**structural**, just like TS. This is the part of PureScript that will feel
most like home:

```purescript
type Person =                          -- a type alias for a record type
  { name :: String
  , age  :: Int
  }

ada :: Person
ada = { name: "Ada", age: 36 }         -- literal: colon, like JS

ada.name                               -- field access: the actual dot
ada { age = 37 }                       -- update: a NEW record, ada untouched
```

```typescript
// TypeScript
interface Person { name: string; age: number }
const ada: Person = { name: "Ada", age: 36 };
ada.name;
{ ...ada, age: 37 };
```

> **Trap:** colon vs equals. **Construction** uses `:` (`{ age: 37 }`, like
> JS); **update** uses `=` (`ada { age = 37 }`). Mixing them up is everyone's
> first record error.

Nested updates have dedicated syntax that beats TS's spread pyramid:

```purescript
user { address { city = "Oslo" } }
```

```typescript
// TypeScript — the pyramid this replaces
{ ...user, address: { ...user.address, city: "Oslo" } };
```

### Row polymorphism: structural typing you can name

A TS function accepting `{ name: string }` happily takes any object with *at
least* a `name` — extra fields ride along. PureScript can express exactly
that, but explicitly, with a **row variable**:

```purescript
greet :: forall r. { name :: String | r } -> String
greet p = "Hello, " <> p.name

greet { name: "Ada" }                        -- ok
greet { name: "Ada", age: 36 }               -- ok — `r` absorbs age
```

Read `{ name :: String | r }` as "a record with a `name :: String` and
*whatever else* (`r`)." Without the `| r`, the record type is exact — extra
fields are a type error. So PureScript gives you both TS behaviors (open
width subtyping vs exact object literals) as an explicit choice per
signature. File `| r` away; you mostly *read* it in library signatures.

⚡ *For Haskell readers: this is the anti-Haskell-records experience —
anonymous, structural, no name clashes, dot access, polymorphic extension.
It's PureScript's single biggest ergonomic win over GHC.*

---

## 3. Sum types: the discriminated union, perfected

A sum type's values are *one of N variants*, each optionally carrying data.
TS's discriminated union is the right mental model — minus the manual tag,
plus real checking:

```purescript
data Shape
  = Circle Number              -- radius
  | Rect Number Number         -- width, height
```

```typescript
// TypeScript — the tag is hand-rolled; in PS the constructor IS the tag
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; width: number; height: number };
```

Consuming one **is** pattern matching, and matching on constructors checks,
narrows, and destructures in one step:

```purescript
area :: Shape -> Number
area (Circle r) = pi * r * r
area (Rect w h) = w * h
```

Here's the part TS can't match: **exhaustiveness is a compile error, not a
lint.** Delete the `Rect` line and the build fails with:

```
A case expression could not be determined to cover all inputs.
The following additional cases are required: Rect _ _
```

Add a `Triangle` variant next year and the compiler hands you a checklist of
every function that needs a new case. TS narrowing only complains if you
use the missed value in a way that happens to conflict; PureScript refuses
to compile the gap, period.

Enums are the no-payload case, and variants mix shapes freely — including
records:

```purescript
data Direction = North | South | East | West

data Event
  = PageView
  | Click { x :: Int, y :: Int }       -- record payload: named fields
  | Purchase String Int
```

The design discipline this enables — mindset shift #3 — is modeling states
so illegal combinations don't exist. Instead of TS's
`{ loading: boolean; data?: Item[]; error?: string }` (which type-checks
`loading && error` nonsense), write the truth:

```purescript
data RequestState
  = Loading
  | Failed String
  | Loaded (Array Item)
```

One value, exactly one state, and every `case` over it must handle all three.

---

## 4. `Maybe` and `Either`: absence and failure, in the type

Two sum types so central they deserve their own section — though after §3
they're visibly *ordinary library types*:

```purescript
data Maybe a = Nothing | Just a        -- Data.Maybe
data Either e a = Left e | Right a     -- Data.Either
```

**`Maybe` replaces `null`/`undefined`.** There is no null in PureScript — a
`Person` is always a whole person, and possible absence is in the type:

```purescript
findPerson :: String -> Array Person -> Maybe Person

greeting :: Maybe Person -> String
greeting (Just p) = "Hello, " <> p.name
greeting Nothing  = "Hello, stranger"
```

```typescript
// TypeScript with strictNullChecks is the same idea — but null still
// sneaks in via any, as, !, JSON.parse, and untyped dependencies.
// Here there is no any, no as, no ! — the check CANNOT be forgotten.
const greeting = (p: Person | null) =>
  p !== null ? `Hello, ${p.name}` : "Hello, stranger";
```

**`Either` is the error union you wish TS had.** `Either e a` is `Left` an
error or `Right` a result (mnemonic: right = correct) — the
`{ ok: false; error: E } | { ok: true; value: A }` result type, standardized:

```purescript
parseAge :: String -> Either String Int
parseAge s = case fromString s of      -- fromString :: String -> Maybe Int (Data.Int)
  Just n
    | n >= 0    -> Right n
    | otherwise -> Left ("negative age: " <> s)
  Nothing       -> Left ("not a number: " <> s)
```

Unlike exceptions, the failure is in the signature, so callers *must*
confront it. What `Either` doesn't yet solve is chaining several failing
steps without a staircase of `case` — that's precisely
[Part 4](04-functor-applicative-monad.md)'s job.

---

## 5. `type` vs `newtype`

```purescript
type Username = String                 -- alias: interchangeable with String
newtype Email = Email String           -- distinct type wrapping String, zero runtime cost
```

- **`type`** is TS's `type Username = string` — documentation only; any
  `String` is accepted. (You met it naming record types in §2 — that's its
  main job.)
- **`newtype`** mints a genuinely *different* type with the same runtime
  representation (the wrapper is erased in the JS output). Use it to stop
  same-shaped values from crossing:

```purescript
sendInvite :: Email -> Username -> Effect Unit
-- arguments can't be swapped, though both are strings underneath

unEmail :: Email -> String
unEmail (Email s) = s                  -- unwrap by pattern matching
```

Rule of thumb: `type` for readability, `newtype` for safety — ids, tokens,
sanitized-vs-raw strings, units.

---

## 6. Type classes: interfaces with explicit, derivable instances

A **type class** declares a capability; an **instance** implements it for
one type — explicitly, separate from the type's definition:

```purescript
class Eq a where
  eq :: a -> a -> Boolean              -- (==) is an alias for eq

instance Eq Direction where
  eq North North = true
  eq South South = true
  eq East  East  = true
  eq West  West  = true
  eq _     _     = false
```

```typescript
// TypeScript's closest shape — but TS interfaces attach to the class
// declaration; PS instances are separate, so you can implement classes
// for types you didn't define (and dispatch can use the RETURN type).
interface Eq<A> { eq(a: A, b: A): boolean }
```

In signatures, a constraint (`=>`) means "for any type with this instance":

```purescript
elem :: forall a. Eq a => a -> Array a -> Boolean
```

— read it like TS's `<A extends Eq>`: the function works generically, *and*
gets to call `eq`. No constraint, no capabilities: a plain `forall a. a -> a`
can only return its argument.

### `derive` — and the `Show` idiom

For the structural classes the compiler can write the instance:

```purescript
data Direction = North | South | East | West

derive instance Eq Direction
derive instance Ord Direction
```

`Show` (the `toString`/`repr` class — `show :: a -> String`) is the one with
a PureScript-specific idiom. Records and primitives show themselves, but a
`data` type can't derive `Show` directly — instead you derive `Generic` (a
structural description of the type) and let `genericShow` do the work:

```purescript
import Data.Generic.Rep (class Generic)
import Data.Show.Generic (genericShow)

derive instance Generic Direction _
instance Show Direction where
  show = genericShow

> show North
"North"
> show { name: "Ada", age: 36 }       -- records just work, no instance needed
"{ age: 36, name: \"Ada\" }"
```

That four-line block (`derive Eq`, `derive Ord`, `derive Generic`,
`Show via genericShow`) is the standard preamble under a `data` declaration
in real code — copy it until it's muscle memory.

⚡ *For Haskell readers: no `deriving` clause on the data declaration —
deriving is standalone, `Show` goes through `Generic`, and since 0.14
instances don't need names.*

The everyday classes to recognize on sight:

| Class | Gives you | TS analogue |
| ----- | --------- | ----------- |
| `Eq` | `==`, `/=` | `===` (by value, recursively) |
| `Ord` | `<`, `compare`, sorting | `Comparable` |
| `Show` | `show` | `JSON.stringify`-ish debug output |
| `Semigroup` | `<>` (combine: string/array concat, …) | `+` for strings, `concat` |
| `Functor`/`Monad` | [Part 4](04-functor-applicative-monad.md) | — |

---

## One-sentence mental model

Records are TS objects with the structural typing made explicit (`| r` for
"and whatever else"), sum types are discriminated unions whose exhaustiveness
the compiler *refuses to compile around*, absence is `Maybe`, failure is
`Either`, distinct concepts get `newtype`s — and a four-line `derive` block
under each `data` type buys equality, ordering, and printing.

---

[← Reading PureScript](01-reading-purescript.md) · [Next: Functions, arrays, purity →](03-functions-arrays-purity.md)
