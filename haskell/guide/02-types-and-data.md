# Part 2 — Types and data

[← Reading Haskell](01-reading-haskell.md) · [Next: Functions, lists, laziness →](03-functions-lists-laziness.md)

Mindset shift #3: in Haskell you design in types first. This part covers the
two ways to build types (records and sum types), the two library types that
replace `null` and error-tuples (`Maybe`, `Either`), and just enough
typeclasses to read `deriving` lines.

One keyword does almost everything here: `data` declares a new type. Recall
from Part 1 that capitalization is enforced — `Circle` is a constructor,
`circle` would be a function.

---

## 1. Product types: records

A record is a type whose values contain *this field AND that field* — a TS
interface or Go struct:

```haskell
data User = User
  { name  :: String
  , age   :: Int
  , email :: String
  }
  deriving (Show, Eq)
```

```typescript
// TypeScript
interface User {
  name: string;
  age: number;
  email: string;
}
```

Note the two `User`s: the first is the **type**, the second is the
**constructor** — the function that builds values
(`User :: String -> Int -> String -> User`). They share a name by convention
(same situation as `newtype PgEnum a = PgEnum a` in
[the deriving-via notes](../typeclasses-newtypes-deriving-via.md#3-newtype--a-zero-cost-wrapper-not-equal-to-itself));
types and values live in separate namespaces, so it never collides.

Construct, read, and "update":

```haskell
alice = User { name = "Alice", age = 30, email = "alice@example.com" }
-- or positionally: User "Alice" 30 "alice@example.com"

λ> age alice                     -- each field is a getter FUNCTION: age :: User -> Int
30
λ> alice { age = 31 }            -- record update: a NEW User, alice untouched
User {name = "Alice", age = 31, email = "alice@example.com"}
```

The update syntax is TS spread, inverted: `alice { age = 31 }` ≈
`{ ...alice, age: 31 }`.

> **Trap:** field accessors are plain top-level functions, so two records in
> one module can't both have a `name` field (the two `name` functions clash).
> Real codebases work around this with prefixes (`userName`), the
> `OverloadedRecordDot` extension (`user.name`), or lenses — all covered in
> [Part 6](06-structuring-programs.md). For now: one record per concept per
> module and you'll never see the problem.

---

## 2. Sum types: the headline feature

A sum type is a type whose values are *this variant OR that variant*. This is
the single biggest data-modeling upgrade over Go, and the closest thing you
know is a TS discriminated union:

```haskell
data Shape
  = Circle Double            -- radius
  | Rect   Double Double     -- width height
  deriving (Show)
```

```typescript
// TypeScript — the discriminant tag is manual; in Haskell the constructor IS the tag
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; width: number; height: number };
```

Each variant is a constructor, possibly carrying data. Consuming a sum type
**is** pattern matching — and matching on constructors is where pattern
matching earns its keep:

```haskell
area :: Shape -> Double
area (Circle r) = pi * r * r
area (Rect w h) = w * h
```

Matching a constructor does three things at once: checks which variant you
have, proves it to the compiler, and binds its fields — the TS
`switch (shape.kind)` + narrowing dance, in one step.

Two properties make this better than what you're used to:

- **Exhaustiveness is checked.** Compile with `-Wall` (do this always) and
  forgetting a variant is a warning at the *definition site* of every function
  that matches — add a `Triangle` variant and the compiler lists every place
  that needs updating. TS narrowing only fails when you try to *use* the
  missed case; Go has nothing.
- **Variants can be a closed set with data attached.** Go's options are `iota`
  constants (no payload, not closed) or an interface plus type switch (open
  set, no exhaustiveness, boilerplate per variant). The thing Go makes painful
  is one line here.

Enums are just the degenerate case — all variants, no payloads:

```haskell
data Direction = North | South | East | West
  deriving (Show, Eq, Enum, Bounded)
```

And the variants of one type can freely mix shapes — payload, record, nothing:

```haskell
data Event
  = PageView                                   -- no data
  | Click Int Int                              -- x y
  | Purchase { item :: String, cents :: Int }  -- record variant
```

**This is mindset shift #3 in practice**: model states as variants and
*illegal states stop compiling*. Instead of
`{ loading: boolean; data?: Items; error?: string }` — which allows
`loading && error` nonsense — write the truth:

```haskell
data RequestState
  = Loading
  | Failed String
  | Loaded [Item]
```

A value is exactly one of these. No flag combinations to keep in your head.

---

## 3. `Maybe` and `Either`: no null, no error-tuples

These two sum types are so central they're worth their own section — but
after §2 you can see they're *ordinary library types*, not language features:

```haskell
data Maybe a = Nothing | Just a            -- defined in the Prelude
data Either e a = Left e | Right a
```

### `Maybe` replaces `null` / `undefined` / `None`

There is no null in Haskell. A `User` is always a whole user. When absence is
possible, it's in the type:

```haskell
findUser :: String -> [User] -> Maybe User
```

The payoff is at the *use site*: you cannot touch the `User` inside without
going through a match, so there is no equivalent of forgetting a null-check —
no `cannot read property of undefined`, ever, anywhere:

```haskell
greeting :: Maybe User -> String
greeting (Just u) = "Hello, " <> name u
greeting Nothing  = "Hello, stranger"
```

```typescript
// TypeScript with strictNullChecks is the same idea —
// but null still leaks in via any, JSON, !, as, non-strict deps.
// In Haskell there is no escape hatch to forget the check.
const greeting = (u: User | null) =>
  u !== null ? `Hello, ${u.name}` : "Hello, stranger";
```

### `Either` replaces Go's `(val, err)`

`Either e a` is "either an error `e` (`Left`) or a result `a` (`Right` —
mnemonic: right = correct)". It's exactly Go's pair of return values, with
the disjunction enforced:

```haskell
parseAge :: String -> Either String Int
parseAge s = case readMaybe s of
  Just n | n >= 0    -> Right n
         | otherwise -> Left ("negative age: " <> s)
  Nothing            -> Left ("not a number: " <> s)
```

```go
// Go — same contract, by convention only
func parseAge(s string) (int, error) {
    n, err := strconv.Atoi(s)
    if err != nil { return 0, fmt.Errorf("not a number: %s", s) }
    if n < 0      { return 0, fmt.Errorf("negative age: %d", n) }
    return n, nil
}
```

The differences: you can't get both, you can't get neither, there's no zero
value standing in for "ignore this half", and ignoring the error variant is
impossible rather than a lint warning. What `Either` does *not* yet solve is
Go's `if err != nil` staircase when chaining several of these — that's
exactly what monads are for, and it's the punchline of
[Part 4](04-functor-applicative-monad.md).

Helpers you'll meet constantly: `fromMaybe` (default a `Maybe`), `maybe` and
`either` (fold each type in one call) — cataloged in
[Part 6](06-structuring-programs.md).

---

## 4. `type` vs `newtype`: aliases and zero-cost wrappers

Two more declaration keywords, both cheaper than `data`:

```haskell
type Username = String              -- alias: interchangeable with String
newtype Email = Email String        -- NEW type: NOT interchangeable with String
```

- **`type`** is TS's `type Username = string` — pure shorthand for
  readability. The compiler sees straight through it; passing a `String`
  where a `Username` is wanted is fine.
- **`newtype`** mints a genuinely distinct type that *wraps* an existing one
  at zero runtime cost. Passing a `String` where an `Email` is wanted is a
  type error; you must wrap (`Email s`) and unwrap explicitly (by matching on
  the constructor). Use it to stop same-shaped values from mixing:

```haskell
sendInvite :: Email -> Username -> IO ()
-- can't swap the args even though both are strings underneath
```

Rule of thumb: `type` for documentation, `newtype` for safety. The deep
dives next door build on exactly this: instance-carrying wrappers in
[the deriving-via notes §3](../typeclasses-newtypes-deriving-via.md), and
tagged ids like `Id User` in [phantom types](../phantom-types.md).

---

## 5. Type variables: generics without the angle brackets

You met lowercase type variables in Part 1 (`length :: [a] -> Int`). The same
works on the *type* side — `data` declarations take parameters:

```haskell
data Maybe a = Nothing | Just a       -- `a` is the type parameter
data Pair a b = Pair a b
```

```typescript
// TypeScript
type Maybe<A> = { tag: "nothing" } | { tag: "just"; value: A };
interface Pair<A, B> { first: A; second: B }
```

Conventions worth internalizing:

- Lowercase in a type = variable; uppercase = concrete. `Maybe a` is generic,
  `Maybe Int` is concrete, and `[a]` is sugar for "list of `a`".
- Type variables with no constraints can't be inspected — a function
  `a -> a` can't add 1 or print; the *only* thing it can do is return its
  argument. Signatures are this informative on purpose: in TS terms, generics
  with no `extends` bound, taken seriously.
- Single letters are idiomatic (`a`, `b`, `e` for error, `f`/`m` for
  contexts — those two become important in [Part 4](04-functor-applicative-monad.md)).

---

## 6. Typeclasses: just enough to read `deriving`

You've seen `deriving (Show, Eq)` on every type above. The full story lives
in [the typeclasses deep-dive](../typeclasses-newtypes-deriving-via.md); the
essentials to read everyday code:

A **typeclass** is an interface a type can implement. The implementation is
called an **instance**, and — unlike Go — it's *explicit and declared
separately from the type*:

```haskell
class Eq a where                       -- the interface
  (==) :: a -> a -> Bool

instance Eq Direction where            -- the implementation for one type
  North == North = True
  South == South = True
  East  == East  = True
  West  == West  = True
  _     == _     = False
```

```go
// Go — closest concept, but structural: any type with the right method
// satisfies the interface implicitly. Haskell instances are opt-in and named.
type Eq interface{ Equals(other any) bool }
```

In signatures, a constraint before `=>` means "any type that implements …"
(read `=>` as "provided that"):

```haskell
elem :: Eq a => a -> [a] -> Bool      -- works for ANY a with an Eq instance
```

That's TS `<A extends Comparable>` / Go's `[T comparable]` — same role.

### The everyday classes

| Class | Gives you | Rough analogue |
| ----- | --------- | -------------- |
| `Eq` | `==`, `/=` | `==` (by value, not reference) |
| `Ord` | `<`, `compare`, `max`, sorting | `Comparable` |
| `Show` | `show` → `String` | `toString` / `repr()` / `%v` |
| `Read` | `read` ← `String` | the inverse of `Show` (prefer `readMaybe`) |
| `Enum`, `Bounded` | `[minBound .. maxBound]`, `succ` | enumerating an enum |
| `Num` | `+`, `*`, literals | "is numeric" |

And **`deriving` asks the compiler to write the instance for you** — the
obvious structural implementation (field-wise equality, constructor-order
comparison, `show` that prints what you typed). For these everyday classes,
deriving is the norm; hand-written instances are for genuinely custom
behavior.

```haskell
data Direction = North | South | East | West
  deriving (Show, Eq, Ord, Enum, Bounded)

λ> [minBound .. maxBound] :: [Direction]
[North,South,East,West]
```

One teaser the deep-dive expands: typeclass methods can dispatch on the
**return** type — `read "42" :: Int` vs `read "42" :: Double` run different
code. No interface in TS/Go/Python can do that, and it's why
parse/decode/deserialize functions in Haskell don't need per-type names.

---

## One-sentence mental model

Records say AND, sum types say OR, and you design by writing the truth:
absence is `Maybe`, failure is `Either`, distinct concepts get distinct
`newtype`s — then pattern matching consumes each shape exhaustively, and
`deriving` hands you equality, ordering, and printing for free.

---

[← Reading Haskell](01-reading-haskell.md) · [Next: Functions, lists, laziness →](03-functions-lists-laziness.md)
