# Phantom types

A **phantom type** is a type parameter that appears on the left of a type
declaration's `=` but in **none of the fields** on the right. It carries no
values and has no runtime representation — it exists only to let the type checker
tell otherwise-identical values apart.

```haskell
newtype Id a = Id Int64
--           ^      ^^^^^
--         the `a`  …never appears here
```

`Id User` and `Id Post` are the same bits at runtime (`Int64`), but **distinct
types** the compiler refuses to mix up. The `a` is "phantom": present in the
type, absent in the data.

---

## 1. The canonical use: tagged identifiers

The everyday problem phantom types solve: every id in your system is an integer,
so the compiler happily lets you pass a user's id where a post's id was wanted.

```haskell
lookupPost :: Int64 -> Int64 -> IO Post   -- (accountId, postId)? (postId, accountId)?
```

Nothing here is checked. Tag the ids and the swap becomes a compile error:

```haskell
newtype Id a = Id { unId :: Int64 }

data User      -- empty "marker" types: never constructed, only used as tags (see §3)
data Post
data Account

type UserId    = Id User
type PostId    = Id Post
type AccountId = Id Account

lookupPost :: AccountId -> PostId -> IO Post
```

Now `lookupPost postId accountId` (arguments flipped) **does not typecheck** —
`Id Post` is not `Id Account`. The signature also *documents itself*: you can read
off exactly which id goes where. Comparing across tags is likewise rejected:

```haskell
(uid :: UserId) == (pid :: PostId)   -- type error: Id User vs Id Post
```

because `(==) :: Eq a => a -> a -> Bool` needs both sides to be the *same* type.

---

## 2. Why a `type` synonym on top

`type UserId = Id User` is not cosmetic. It does two jobs:

- **Readability** — signatures say `UserId`, not `Id User`. Callers never need to
  think about the `Id a` machinery.
- **Decoupling** — code that uses `UserId` never names the bare tag `User`, so a
  marker tag named `User` can't collide with an unrelated `User` *record* elsewhere
  in the program. (Keep the markers unexported and only export the synonyms, and the
  collision is impossible.)

Underneath, `UserId` is still `Id User`, so all the type-distinctness from §1 holds.

---

## 3. Where the tag types come from

The phantom needs *something* to stand in for `a`. Three common choices:

| Approach | Looks like | When |
| -------- | ---------- | ---- |
| **Empty marker types** | `data User` (no constructors) | Default. Standalone, zero deps, can't be constructed by accident. Needs `EmptyDataDecls` (on by default in `GHC2021`). |
| **Reuse existing types** | `type UserId = Id User` where `User` is the real record | DRY when the type is already in scope — but if the id module must *import* every record just to tag with it, you can get **import cycles**. Markers sidestep that. |
| **`DataKinds`-promoted** | `Id 'User` from `data Entity = User \| Post` | When you want a *closed* set the compiler enforces — only the listed tags are well-kinded. More machinery; reach for it when an open set of marker types is too loose. |

The empty-marker approach is the usual starting point: a tag is just a name at the
type level, and `data User` with no constructors is the minimal way to mint one.

---

## 4. Deriving *through* a phantom

This is the payoff that makes the pattern cheap, and it leans on **roles**.

Because `a` never appears in `Id`'s representation, its role is **phantom**, which
means `Id a` and `Id b` are `Coercible` for *any* `a`, `b`. Two consequences:

```haskell
newtype Id a = Id { unId :: Int64 }
    deriving stock   (Show, Eq, Ord)
    deriving newtype (FromField, ToField, FromJSON, ToJSON, FromHttpApiData, ToHttpApiData)
```

- **`deriving newtype (…)` is written once and covers every tag.** Each derived
  instance is polymorphic in the tag — `instance FromField (Id a)` — because the
  underlying `Int64` instance is reused by coercion regardless of `a`. One block
  serves `UserId`, `PostId`, and every future id.
- **`deriving stock (Eq, Ord)` adds no spurious constraint.** Stock deriving only
  constrains type variables that appear in a *field*. `a` appears in none, so you
  get `instance Eq (Id a)` with no `Eq a` context — exactly right, since there's no
  `a` value to compare.

> **The flip side of phantom roles — a safety caveat.** Phantom role also means
> `coerce (Id 5 :: UserId) :: PostId` compiles. The tag protects you from
> *accidental* mixing through ordinary functions, but it is not a hard wall: anyone
> who calls `coerce`, or uses the raw `Id` constructor, can relabel an id freely. The
> guarantee is "good-faith" — strong against slips, defeated by deliberate coercion.
> Keep the constructor's use disciplined (smart constructors, limited exports) if the
> distinction matters.

---

## 5. Tradeoffs and limitations

- **`Show` (and runtime in general) can't see the tag.** Since the phantom has no
  runtime representation, `show (Id 5 :: UserId)` produces `Id 5` — nothing
  distinguishes it from a `PostId`. Recoverable with a `Typeable`-based custom
  `Show`, but not for free.
- **Construction goes through the real constructor, not the synonym.** `UserId` is a
  *type* synonym, not a data constructor, so you write `Id 5` and let inference pin
  the tag from context (`lookupUser (Id 5)`), or annotate (`Id 5 :: UserId`).
- **One accessor for all tags.** `unId :: Id a -> Int64` unwraps every id, instead of
  a separate `unUserId`, `unPostId`, … (A small win the multi-newtype alternative
  doesn't get.)

The main thing you give up versus *one explicit `newtype` per entity*
(`newtype UserId = UserId Int64`, repeated N times) is the nicer `Show`
(`UserId 5`) and the obvious `UserId 5` constructor — in exchange for write-once
derivation and a single unwrap. Same call-site readability either way.

---

## 6. Beyond ids: other classic phantom patterns

Tagged ids are the gateway example, but the same "type-level label, no runtime
data" idea shows up widely:

**Typestate** — encode an object's state in its type so illegal operations don't
compile:

```haskell
data Open
data Closed
newtype Door s = Door FilePath        -- s :: Open | Closed

newDoor     :: FilePath -> Door Closed
open        :: Door Closed -> Door Open
close       :: Door Open   -> Door Closed
walkThrough :: Door Open   -> IO ()
```

`walkThrough` accepts only a `Door Open`; `open (open d)` is a type error. The
state machine is checked at compile time, and `s` is never a value.

**Units of measure** — stop dimensionally nonsensical arithmetic:

```haskell
newtype Quantity u = Quantity Double   -- u :: Meters | Seconds | …
-- (+) on Quantity Meters is fine; adding Quantity Meters to Quantity Seconds won't typecheck
```

**Validation / provenance tags** — distinguish "raw" from "checked" data so a
function can demand the checked kind: `Tagged Sanitized Text` vs `Tagged Raw Text`.
The `tagged` library generalizes exactly this with `newtype Tagged s b = Tagged b`,
where `s` is a pure phantom label.

**The next step up.** When you want the *value* to actually determine the phantom
(not just thread it through), phantom types graduate into **GADTs** — there the
constructor can refine the type index — and **`DataKinds`** promotes your tags to a
real kind so only intended labels are well-formed. Plain phantom types are the
lightweight floor of that ladder.

---

## One-sentence mental model

A phantom type parameter is a **compile-time label with no runtime existence**: it
costs nothing, disappears when the program runs, and its entire job is to make the
type checker treat same-shaped values as different — catching swapped ids, illegal
state transitions, and mismatched units before the code ever runs.

---

## Source

The term comes from Leijen & Meijer's **"Domain-Specific Embedded Compilers"**
(1999), which used phantom types to give a typed embedded SQL/query DSL. The idea
underpins much of the `safe-by-construction` style in modern Haskell, and the
standard `Data.Tagged` (`tagged` package) and `Data.Functor.Const` are minimal
phantom carriers. Roles — the mechanism that makes phantom parameters coercible and
deriving free — were introduced in GHC 7.8 (Breitner et al., "Safe Coercions").
