# Typeclasses, newtypes, and `deriving via`

Notes from understanding a real module (`FabulaTabula.Db.Enum`) that uses all
three together. The module's job: let a plain Haskell enum (e.g.
`data EntityType = Character | Location | Faction`) be read/written to Postgres
and JSON by writing the string mapping **once**, instead of hand-writing four
typeclass instances per enum.

The reference code this doc explains:

```haskell
-- The ONE thing each enum must provide: its mapping to/from the DB text.
class DbEnum a where
    toDbText   :: a -> Text
    fromDbText :: Text -> Maybe a

-- A carrier newtype whose only job is to host the four instances below.
newtype PgEnum a = PgEnum a

instance (DbEnum a) => ToField (PgEnum a) where
    toField (PgEnum x) = toField (toDbText x)

instance (DbEnum a, Typeable a) => FromField (PgEnum a) where
    fromField f mb = do
        t <- fromField f mb
        maybe
            (returnError ConversionFailed f ("bad enum value: " <> show t))
            (pure . PgEnum)
            (fromDbText t)

instance (DbEnum a) => ToJSON   (PgEnum a) where
    toJSON (PgEnum x) = toJSON (toDbText x)

instance (DbEnum a) => FromJSON (PgEnum a) where
    parseJSON = withText "PgEnum" $ \t ->
        maybe (fail ("bad enum value: " <> show t)) (pure . PgEnum) (fromDbText t)
```

And how an enum uses it:

```haskell
data EntityType = Character | Location | Faction
    deriving stock (Show, Eq, Generic)
    deriving (ToField, FromField, ToJSON, FromJSON) via PgEnum EntityType

instance DbEnum EntityType where
    toDbText Character = "character"
    toDbText Location  = "location"
    toDbText Faction   = "faction"
    fromDbText "character" = Just Character
    fromDbText "location"  = Just Location
    fromDbText "faction"   = Just Faction
    fromDbText _           = Nothing
```

---

## 1. `class` is an interface, not an OOP class

A Haskell `class` declares a **contract** — a set of functions a type must
provide. Despite the keyword, it is **not** the OOP `class`: it holds no data,
has no constructor, and is never instantiated into an object.

| Haskell                              | Closest OOP analogue                         |
| ------------------------------------ | -------------------------------------------- |
| `class DbEnum a where …`             | an **interface** / Rust trait / Swift protocol |
| `instance DbEnum EntityType where …` | `class EntityType implements DbEnum`         |
| `toDbText`, `fromDbText`             | the interface's method signatures            |

Read `class DbEnum a where …` as: *"To be a `DbEnum`, a type `a` must supply
`toDbText` and `fromDbText`."* The `a` is a placeholder for whatever type
implements it. An `instance` is the implementation for one concrete type.

### The superpower: dispatch on return type

This is where the interface analogy **breaks** — and why typeclasses are more
powerful than OOP interfaces. Look at the two signatures:

```haskell
toDbText   :: a -> Text         -- takes an `a`  (like `this`/`self`)
fromDbText :: Text -> Maybe a   -- takes Text, returns `a` — NO `a` argument!
```

`fromDbText` has no `a` parameter. In Java/Python an interface method always
dispatches on the object it's called on (`this`). With no object to dispatch on,
how does Haskell pick the implementation? **It dispatches on the expected return
type at the call site:**

```haskell
fromDbText "character" :: Maybe EntityType   -- runs EntityType's impl
fromDbText "attribute" :: Maybe FieldKind    -- runs FieldKind's impl
```

Same function, same argument type, different implementation chosen by *what type
you want back*. OOP interfaces fundamentally cannot do this. This is exactly why
`fromDbText` works as a "parse / factory" function inside the `FromField` and
`FromJSON` instances: those contexts already know they want a `PgEnum a` back, so
the correct `fromDbText` is selected automatically.

---

## 2. `=>` is the constraint arrow — read it as "provided that"

`=>` introduces a **constraint**: a requirement that must hold for an instance or
function to apply. Read it as *"provided that"* or *"implies."*

```haskell
instance (DbEnum a) => ToField (PgEnum a) where
```

*"`PgEnum a` is a `ToField` — **provided that** `a` is a `DbEnum`."* Left of `=>`
is the **requirement**; right of `=>` is what you **get** when it holds.

This is what makes the body legal:

```haskell
toField (PgEnum x) = toField (toDbText x)
--                            ^^^^^^^^ allowed ONLY because (DbEnum a) => guarantees
--                                     this `a` has a toDbText
```

Strip the `(DbEnum a) =>` and GHC rejects the body: nothing would promise `a`
has a `toDbText`.

Two things to file away:

- **`->` vs `=>`.** `->` is the value-level function arrow: value in, value out
  (`a -> Text`). `=>` is the constraint arrow: *evidence a constraint holds* in,
  the constrained thing out. Different arrows, different worlds.
- **Multiple constraints** are comma-separated inside parens, and *all* must
  hold: `(DbEnum a, Typeable a) =>`. (`Typeable a` is needed here only because
  `returnError` embeds the type's name in its error message.)

---

## 3. `newtype` — a zero-cost wrapper, not "equal to itself"

```haskell
newtype PgEnum a = PgEnum a
--      ^^^^^^^^   ^^^^^^^^
--      (A) type   (B) data
--      constructor    constructor
```

It looks like `x = x`, but the two `PgEnum`s live in **different namespaces** and
only share a name (Haskell convention when there's one constructor):

- **(A)** `PgEnum a`, left of `=`, is a **type constructor** — it lives in the
  type world. `PgEnum EntityType` is a *type*.
- **(B)** `PgEnum a`, right of `=`, is a **data constructor** — a function
  `a -> PgEnum a` that wraps one value. `PgEnum Character` is a *value*.

Types and values are separate namespaces, so the repetition never collides. The
line means: *"a new type `PgEnum a` whose single constructor (also `PgEnum`)
holds exactly one value of type `a`."* A box with one thing in it.

### Why wrap something in itself? Two facts that make the whole pattern work

> `PgEnum EntityType` is a **different type** from `EntityType` (the type checker
> treats them as distinct), but at **runtime it is the identical bits** (the
> wrapper is erased).

- **Different type → can carry different instances.** `EntityType` alone has no
  `ToField`. `PgEnum a` does. Wrapping is how you reach a *different* set of
  instances than the bare type has. The wrapper exists *only* as a hook to hang
  instances on.
- **Same bits → wrapping is free.** This is why it's `newtype`, not `data`. A
  `newtype` is restricted to exactly one constructor with exactly one field, and
  in return the compiler guarantees **zero runtime cost**: `PgEnum Character` and
  `Character` are represented identically in memory. A `data` wrapper would add a
  real box + pointer indirection.

So: *same representation* (free coercion) + *distinct type* (own instances) =
borrow behavior for nothing.

---

## 4. `deriving via` — tie it together

```haskell
data EntityType = Character | Location | Faction
    deriving stock (Show, Eq, Generic)
    deriving (ToField, FromField, ToJSON, FromJSON) via PgEnum EntityType
```

`deriving C via T` tells GHC: *"to get `C EntityType`, coerce `EntityType` to
`T` (here `PgEnum EntityType`), use **its** `C` instance, coerce back."* Because
`EntityType` and `PgEnum EntityType` are the same bits (§3), the coercion is
free. So `EntityType` inherits all four instances written once on `PgEnum`, and
the only per-enum code you write is the `DbEnum` instance (the actual string
mapping, which is irreducible — something has to say `Character` ↔ `"character"`).

The `deriving stock` on the line above is needed because, with `DerivingVia`
turned on, a bare `deriving` is ambiguous (instances can come from several
mechanisms). The strategy keywords disambiguate:

- `stock` — GHC's built-in generators (`Show`, `Eq`, `Generic`, …)
- `newtype` — reuse the wrapped type's instance (a.k.a. GeneralizedNewtypeDeriving)
- `anyclass` — an empty instance using the class's default methods
- `via` — borrow a chosen representationally-equal type's instance

**Key insight:** `deriving newtype X` is just the special case of
`deriving X via <the-wrapped-type>`. `deriving via` generalizes it from "the one
type you happen to wrap" to "*any* type of the same representation you choose."

### Why you can't skip the wrapper with one blanket instance

The obvious-but-illegal shortcut is `instance ToField a where …` for "all my
enums." It doesn't work: a blanket instance overlaps with **every** existing
`ToField` instance (`Int64`, `Text`, …), and Haskell has no way to scope it to
"only my enum types." The `PgEnum` wrapper is the escape hatch — instances on
`PgEnum a` apply only to things explicitly deriving via it.

---

## One-sentence mental model

`DbEnum` is the contract (to/from `Text`); `PgEnum` is a free wrapper that exists
only to host instances; the four instances say *"provided a type is a `DbEnum`,
its `PgEnum` wrapper knows how to talk to Postgres and JSON"*; and
`deriving … via PgEnum T` lets each enum borrow those instances at zero runtime
cost.

---

## Source

`deriving via` comes from the paper **"Deriving Via: or, How to Turn Hand-Written
Instances into an Anti-Pattern"** (Blöndal, Löh, Scott; Haskell Symposium 2018),
which became GHC's `DerivingVia` extension in 8.6. The standard library now ships
adapter newtypes built on it — e.g. `Generically` in `GHC.Generics` (base ≥ 4.17),
and the older `Sum`/`Product`/`Min`/`Max` monoid newtypes are the same idea
predating the feature.
