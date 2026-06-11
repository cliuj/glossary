# Part 4 — Functor → Applicative → Monad

[← Functions, arrays, purity](03-functions-arrays-purity.md) · [Next: Effect and Aff →](05-effect-and-aff.md)

Part 2 left a debt: `Either` makes failure visible, but chaining several
failing steps threatened a staircase of nested `case`. This part pays it
off. The fix is three ordinary type classes of increasing power — and the
last one is the M-word, which you have been using for years under the name
`flatMap` and `await`.

Strategy: intuition first, laws never. By the end, `Monad` should feel about
as exotic as `Iterable`.

---

## 1. The shape everything shares: a value in a context

Line up the types from the previous parts as one family:

| Type | The context it adds |
| ---- | ------------------- |
| `Maybe a` | an `a`... or absence |
| `Either e a` | an `a`... or an error `e` |
| `Array a` | any number of `a`s |
| `Effect a` / `Aff a` | an `a`... produced by running side effects ([Part 5](05-effect-and-aff.md)) |

Each is "an `a`, plus a story about how you get it." The three classes
answer one escalating question: **you have a plain function — how do you use
it on a value wrapped in a story?**

---

## 2. Functor: map over the context

You already do this daily — `array.map` applies a plain function inside the
"many values" story. PureScript's `map` is that, generalized to *every*
context (arrays are just one instance):

```purescript
class Functor f where
  map :: forall a b. (a -> b) -> f a -> f b
```

```purescript
> map (_ + 1) [1, 2, 3]            -- the Array instance is .map
[2,3,4]
> map (_ + 1) (Just 5)             -- apply inside, keep the wrapper
(Just 6)
> map (_ + 1) Nothing              -- nothing inside, nothing to do
Nothing
> map (_ + 1) (Right 5)            -- Either maps the success side...
(Right 6)
> map (_ + 1) (Left "boom")        -- ...errors pass through untouched
(Left "boom")
```

Over `Maybe`, `map` is optional chaining:

```typescript
// TypeScript — "apply .name only if non-null, else stay null"
const userName = user?.name;
```

```purescript
userName = map _.name maybeUser    -- Maybe User -> Maybe String
```

The operator `<$>` is infix `map`, and it's everywhere — read `f <$> x` as
"`f`, mapped over `x`":

```purescript
> String.length <$> Just "hello"
(Just 5)
```

What `map` can't do: use a function whose *result is itself wrapped*, or
combine *two* wrapped values. Climb.

---

## 3. Applicative: combine several contexts

The motivating case is building a value from several wrapped parts.
Currying meets `map`: `mkPerson <$> validName s` gives a `Maybe`-wrapped
*partially applied function*, and `Applicative`'s `<*>` keeps applying it:

```purescript
class Functor f => Applicative f where
  pure  :: forall a. a -> f a                            -- wrap a plain value
  apply :: forall a b. f (a -> b) -> f a -> f b          -- the <*> operator
```

The idiom — a constructor applied across wrapped arguments, all-or-nothing:

```purescript
type Person = { name :: String, age :: Int }

mkPerson :: String -> String -> Maybe Person
mkPerson name ageStr =
  { name: _, age: _ } <$> validName name <*> Int.fromString ageStr
--  ^ record-literal section: a 2-arg function filling the holes in order
--    if BOTH succeed → Just a person; if EITHER fails → Nothing
```

```typescript
// TypeScript — the same "all or nothing", spelled by hand
const mkPerson = (name: string, ageStr: string): Person | null => {
  const n = validName(name);
  const a = parseInt2(ageStr);
  return n !== null && a !== null ? { name: n, age: a } : null;
};
```

Read `f <$> x <*> y <*> z` as `f(x, y, z)` *where every argument is wrapped
and the result exists only if all of them do*. The arguments are independent
— none can look at another's result. That independence is exactly what
`Applicative` expresses, and its limit: when step 2 needs step 1's *value*,
you need the last rung.

---

## 4. Monad: sequence steps that depend on each other

The real-world shape: find a user, *then with that user* find their manager,
*then with the manager* get an email — each step needs the previous result,
each can fail:

```purescript
findUser  :: String -> Maybe User
managerOf :: User -> Maybe User
workEmail :: User -> Maybe String
```

Chained by hand, the staircase:

```purescript
managerEmail :: String -> Maybe String
managerEmail id =
  case findUser id of
    Nothing -> Nothing
    Just u  -> case managerOf u of
      Nothing -> Nothing
      Just m  -> workEmail m
```

Every floor repeats one move: *if absent, stop; if present, unwrap and feed
the next step.* `Monad` names that move `bind`, operator `>>=`:

```purescript
class Applicative m => Monad m where        -- (slightly simplified)
  bind :: forall a b. m a -> (a -> m b) -> m b
  --      ^ wrapped a    ^ next step: takes the UNWRAPPED value,
  --                       returns a new wrapped result

managerEmail id = findUser id >>= managerOf >>= workEmail
```

The whole staircase, because `Maybe`'s `bind` *is* "stop on Nothing, unwrap
on Just," written once in the instance. Compare the three signatures — the
ladder is just "where may the function put its result":

```purescript
map   :: (a ->   b) -> f a -> f b     -- plain result
apply :: f (a -> b) -> f a -> f b     -- the function itself is wrapped
bind  :: m a -> (a -> m b) -> m b     -- the step CHOOSES the next context
```

### do-notation is sugar for `>>=`

Chains of binds with lambdas get noisy, so there's syntax — and it's *only*
syntax:

```purescript
managerEmail id = do           --  managerEmail id =
  u <- findUser id             --    findUser id >>= \u ->
  m <- managerOf u             --    managerOf u >>= \m ->
  workEmail m                  --    workEmail m
```

Reading rules for any do-block:

- `x <- step` — run the step, bind its unwrapped result (it's `>>=`
  underneath, **not** assignment).
- `let x = expr` — name a *pure* value (no `in` needed inside `do`).
- the last line is the whole block's result and must be in the monad — wrap
  plain values with `pure`. ⚡ *There is no `return` alias; it's `pure`, always.*
- a line with no `<-` runs the step and discards its result.

> **Trap:** mixing up `<-` and `let` in a do-block. `<-` takes a wrapped
> `m a` on its right; `let` takes a plain `a`. If the compiler says it
> couldn't match `Int` with `Maybe Int` (or the reverse) on a do-line,
> you've used one where the other belongs.

---

## 5. The workhorse monads

Same do-syntax, different superpower — the instance decides what "then"
means:

**`Maybe` — first `Nothing` wins.** The do-block above is
`findUser(id)?.manager?.workEmail` generalized beyond property access: any
step can be "stop here if absent."

**`Either` — first `Left` wins, and says why.** Part 2's debt, paid. The
hand-rolled TS result-chain and its PureScript translation:

```typescript
// TypeScript with a Result type — every step needs the guard
function process(input: string): Result<Order, Err> {
  const parsed = parse(input);
  if (!parsed.ok) return parsed;
  const validated = validate(parsed.value);
  if (!validated.ok) return validated;
  return price(validated.value);
}
```

```purescript
process :: String -> Either Err Order
process input = do
  parsed    <- parse input
  validated <- validate parsed
  price validated
```

Each `if (!x.ok) return x` is `Either`'s `bind`, written once in the library
instead of after every call. Same semantics — short-circuit on first error —
but a check that isn't your code can't be forgotten by your code.

**`Array` — try every combination.** You already know this monad by name:
**`bind` for arrays is `flatMap`.**

```typescript
// TypeScript
[1, 2, 3].flatMap(x => ["a", "b"].map(y => [x, y]));
```

```purescript
pairs = do                     -- a do-block over arrays is nested loops
  x <- [1, 2, 3]
  y <- ["a", "b"]
  pure [x, y]
```

---

## 6. Demystification

> **A monad is a type with `pure` and `bind`. That's the whole thing.** Not
> a container, not a burrito — an interface with two methods and the
> convention that `bind` means "then." The mystique exists because the
> interface is unusually abstract: its instances (absence, failure,
> multiplicity, effects) share no surface resemblance, only the sequencing
> shape. Don't chase "monads in general" — know `Maybe`, `Either`, and
> `Array` well, and the abstraction reveals itself.

And the kicker for a TS developer — you've written monadic code daily for
years: **`async/await` is do-notation for Promises.** `await` unwraps like
`<-`, the implicit "then" between statements is `bind`, `return` inside
`async` is `pure`:

```typescript
// TypeScript                        -- PureScript (Aff, in Part 5)
async function f() {                 -- f = do
  const a = await stepOne();         --   a <- stepOne
  const b = await stepTwo(a);        --   b <- stepTwo a
  return a + b;                      --   pure (a + b)
}
```

The analogy is imperfect (Promises start running when created and
auto-flatten), but the *experience* — straight-line code, the wrapper
threads itself — is exactly what do-notation gives every context on this
page. Part 5's `Aff` makes the correspondence nearly literal.

---

## 7. Everyday combinators

Three you'll meet in every file, here for recognition and used for real in
[Part 5](05-effect-and-aff.md):

```purescript
traverse :: (a -> m b) -> Array a -> m (Array b)   -- Data.Traversable
```

"`map` where the function has a context — and the contexts combine":
`traverse Int.fromString ["1","2"]` is `Just [1,2]`; any failure makes the
whole thing `Nothing`. One function = "validate every field," "fetch every
id," "parse every row." (`traverse_` is the same, discarding results —
for effects.)

```purescript
for_ :: Array a -> (a -> m Unit) -> m Unit         -- Data.Foldable
```

`traverse_` with the arguments flipped — reads like a foreach:
`for_ users \u -> log u.name`.

```purescript
when, unless :: Boolean -> m Unit -> m Unit        -- Prelude
```

Conditional *effects* without an else branch: `when verbose (log msg)`.

---

## One-sentence mental model

Functor, Applicative, and Monad are three grades of "use a plain function on
wrapped values" — `map` reaches inside one wrapper, `<*>` combines several
independent ones, `>>=` lets each step choose the next from the last result
— do-notation is sugar for `>>=` chains, the instance defines what "then"
means (stop at `Nothing`, keep the first `Left`, `flatMap` over arrays), and
async/await was this all along.

---

[← Functions, arrays, purity](03-functions-arrays-purity.md) · [Next: Effect and Aff →](05-effect-and-aff.md)
