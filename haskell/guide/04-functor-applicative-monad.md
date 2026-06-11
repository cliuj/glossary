# Part 4 — Functor → Applicative → Monad

[← Functions, lists, laziness](03-functions-lists-laziness.md) · [Next: IO and effects →](05-io-and-effects.md)

Part 2 left a debt unpaid: `Either` gives you Go's `(val, err)` contract, but
chaining several `Either`-returning steps looked like it would be Go's
`if err != nil` staircase with `case` instead of `if`. This part pays the
debt. The fix is not a language feature — it's three ordinary typeclasses of
increasing power, and the last one is the infamous M-word.

Strategy: intuition first, laws never (they exist; you don't need them to
read or write code). Nothing here is magic — by the end, `Monad` should feel
about as exotic as `Iterable`.

---

## 1. The shape everything shares: a value in a context

Look at the types from the last two parts as one family:

| Type | The context it adds |
| ---- | ------------------- |
| `Maybe a` | an `a`... or absence |
| `Either e a` | an `a`... or an error `e` |
| `[a]` | any number of `a`s |
| `IO a` | an `a`... obtained by performing side effects ([Part 5](05-io-and-effects.md)) |

Each is "an `a`, plus a story about how you get it." The three classes in
this part answer one escalating question: **you have a plain function — how
do you use it on a value that's wrapped in a story?** (This is where the
`f`/`m` type-variable convention from Part 2 §5 lands: `f` for a context as
a *functor*, `m` as a *monad*.)

---

## 2. Functor: map a function over the context

You already do this. JS `array.map` applies a plain function inside the
"many values" context. `Functor` is that idea, generalized to *any* context:

```haskell
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```

```haskell
λ> fmap (+1) [1, 2, 3]          -- the list instance IS map
[2,3,4]
λ> fmap (+1) (Just 5)           -- absence: apply inside, keep the wrapper
Just 6
λ> fmap (+1) Nothing            -- nothing inside, nothing to do
Nothing
λ> fmap (+1) (Right 5 :: Either String Int)
Right 6
λ> fmap (+1) (Left "boom" :: Either String Int)   -- errors pass through untouched
Left "boom"
```

`fmap` over `Maybe` is TypeScript's optional chaining:

```typescript
// TypeScript — "apply .name only if non-null, else stay null"
const userName = user?.name;
```

```haskell
userName = fmap name maybeUser     -- Maybe User -> Maybe String
```

The operator `<$>` is just infix `fmap`, and it's everywhere — read
`f <$> x` as "`f`, mapped over `x`":

```haskell
λ> length <$> Just "hello"
Just 5
```

What `fmap` can't do: the function must be pure and one-argument-shaped. Two
wrapped values, or a step that can *itself* fail? Climb the ladder.

---

## 3. Applicative: combine several contexts

The motivating case: building a value from **several** wrapped parts.
`User <$> parseName s` gets you a `Maybe (Int -> User)` — a *partially
applied constructor stuck inside the context* (currying from Part 1, meeting
`fmap`). `Applicative` provides the operator that keeps applying:

```haskell
class Functor f => Applicative f where
  pure  :: a -> f a                  -- wrap a plain value: the "trivial story"
  (<*>) :: f (a -> b) -> f a -> f b  -- apply a wrapped function to a wrapped value
```

The idiom to internalize — constructor applied across wrapped arguments:

```haskell
data User = User { name :: String, age :: Int }

mkUser :: String -> String -> Maybe User
mkUser n a = User <$> nonEmpty n <*> readMaybe a
--           ^ if BOTH succeed, you get Just (User ...);
--             if EITHER is Nothing, the whole thing is Nothing.
```

```typescript
// TypeScript — the same "all or nothing", spelled by hand:
const mkUser = (n: string, a: string): User | null => {
  const name = nonEmpty(n);
  const age = parseAge(a);
  return name !== null && age !== null ? { name, age } : null;
};
```

Read `f <$> x <*> y <*> z` as `f(x, y, z)` *where every argument is wrapped
and the result only exists if they all do*. The arguments are independent —
none looks at another's result. That independence is exactly what
`Applicative` can express and no more: the *shape* of the computation is
fixed up front. When step 2 needs step 1's **value**, you need the next rung.

---

## 4. Monad: sequence steps where each depends on the last

The real-world shape: look up a user, *then with that user* find their
manager, *then with the manager* get the email. Each step needs the previous
**result**, and each step can fail:

```haskell
lookupUser    :: String -> Maybe User
managerOf     :: User -> Maybe User
workEmail     :: User -> Maybe String
```

Chaining by hand is the nested-case staircase:

```haskell
managerEmail :: String -> Maybe String
managerEmail uid =
  case lookupUser uid of
    Nothing -> Nothing
    Just u  -> case managerOf u of
      Nothing -> Nothing
      Just m  -> workEmail m
```

Every floor of the staircase is the same move: *if absent, stop; if present,
unwrap and feed the next step.* `Monad` names that move `>>=` ("bind"):

```haskell
class Applicative m => Monad m where
  (>>=) :: m a -> (a -> m b) -> m b
  --       ^ a wrapped a   ^ the next step: takes the UNWRAPPED value,
  --                         returns a new wrapped result
```

```haskell
managerEmail uid = lookupUser uid >>= managerOf >>= workEmail
```

That's the entire pattern-matching staircase, because `Maybe`'s `>>=` *is*
the "stop on Nothing, unwrap on Just" logic, written once in the instance:

```haskell
Nothing >>= _    = Nothing
Just x  >>= next = next x
```

Compare the signatures and the ladder is just "where may the function put
its result":

```haskell
fmap  :: (a ->   b) -> f a -> f b   -- plain result
(<*>) :: f (a -> b) -> f a -> f b   -- the function itself is wrapped
(>>=) :: m a -> (a -> m b) -> m b   -- the step CHOOSES the next context
```

### do-notation is sugar for `>>=`

Chains of `>>=` with lambdas get noisy, so Haskell has syntax for them —
and it's *only* syntax. Same `managerEmail`, plus the desugaring:

```haskell
managerEmail uid = do            --  managerEmail uid =
  u <- lookupUser uid            --    lookupUser uid >>= \u ->
  m <- managerOf u               --    managerOf u    >>= \m ->
  workEmail m                    --    workEmail m
```

Reading rules for any do-block:

- `x <- action` — run the step, bind its unwrapped result to `x`
  (it's `>>=` underneath; **not** assignment).
- `let x = expr` — name a *pure* value, no context involved (no `in` needed
  here — the one place `let` drops it).
- the last line is the result of the whole block, and must be in the monad
  (wrap plain values with `pure`).
- a line with no `<-` runs the step and ignores its value.

> **Trap:** mixing up `<-` and `let` inside `do`. `<-` takes `m a` on the
> right; `let` takes a plain `a`. If GHC complains it
> `Couldn't match Int with Maybe Int` (or vice versa) on a do-line, you've
> used one where the other belongs.

---

## 5. The workhorse monads

Same `do` syntax, different superpower per type — the instance decides what
`>>=` *means*:

**`Maybe` — first `Nothing` wins.** The optional-chaining monad: the
do-block above is `lookupUser(uid)?.manager?.workEmail` with user-defined
steps, not just property access.

**`Either` — first `Left` wins, and tells you why.** Here is Part 2's
cliffhanger resolved. The Go staircase and its Haskell translation:

```go
// Go
func process(input string) (Order, error) {
    parsed, err := parse(input)
    if err != nil { return Order{}, err }
    validated, err := validate(parsed)
    if err != nil { return Order{}, err }
    priced, err := price(validated)
    if err != nil { return Order{}, err }
    return priced, nil
}
```

```haskell
process :: String -> Either Err Order
process input = do
  parsed    <- parse input
  validated <- validate parsed
  price validated
```

Every `if err != nil { return ..., err }` is the `Either` instance's `>>=`,
written once in the library instead of after every call. The semantics are
identical — short-circuit on first error, pass the error up — but it's
*impossible to forget a check*, because the check isn't your code.

**Lists — try every combination.** `>>=` for lists runs the rest of the
block once per element (it's `flatMap`); a do-block over lists is a nested
loop, and a comprehension is sugar over exactly this:

```haskell
pairs = do          -- same as: [(x, y) | x <- [1, 2, 3], y <- "ab"]
  x <- [1, 2, 3]
  y <- "ab"
  pure (x, y)
```

---

## 6. Demystification

> **A monad is a type with `pure` and `>>=`. That's the whole thing.** Not a
> container, not a burrito, not category-theory homework — an interface with
> two methods and the convention that `>>=` means "then". The mystique
> exists because the interface is *unusually* abstract — its instances
> (absence, errors, multiplicity, I/O) share no surface resemblance, only
> the sequencing shape. You don't need to "get" monads in general; know
> three instances well and the abstraction reveals itself.

If you've written modern TypeScript you have already used monadic
sequencing daily: **`async/await` is do-notation for Promises.** `await`
unwraps like `<-`; the implicit "then" between statements is `>>=`; an
`async` function's return is `pure`. The analogy is imperfect (Promises
eagerly execute and merge nested layers), but the *experience* — "write
straight-line code, the wrapper threads itself" — is exactly what do-notation
generalizes to every context on this page.

```typescript
// TypeScript                        -- Haskell
async function f() {                 -- f = do
  const a = await stepOne();         --   a <- stepOne
  const b = await stepTwo(a);        --   b <- stepTwo a
  return a + b;                      --   pure (a + b)
}
```

---

## 7. Everyday combinators

A handful of functions built on this machinery appear in every file; they're
covered by recognition here and used for real in [Part 5](05-io-and-effects.md):

```haskell
traverse :: (a -> m b) -> [a] -> m [b]
```

"`map`, where the function has a context — and the contexts combine."
`traverse parseAge ["1","2"]` is `Just [1,2]`; if any element fails, the
whole thing is `Nothing`. This one function is "validate a whole list",
"fetch every id", "parse every row". (`mapM` is the same thing by an older
name; `mapM_` discards results — common for effects.)

```haskell
when   :: Monad m => Bool -> m () -> m ()    -- run the action only if True
unless :: Monad m => Bool -> m () -> m ()    -- ...only if False
```

Conditional *effects* without an `else` branch — `when verbose (logLine s)`.

---

## One-sentence mental model

Functor, Applicative, and Monad are three grades of the same question — *use
a plain function on wrapped values* — where `fmap` maps inside one wrapper,
`<*>` combines several independent ones, and `>>=` lets each step choose the
next based on the last result; do-notation is sugar for `>>=`-chains, the
instance defines what "then" means (stop at `Nothing`, carry the first
`Left`, loop over a list), and async/await was this all along.

---

[← Functions, lists, laziness](03-functions-lists-laziness.md) · [Next: IO and effects →](05-io-and-effects.md)
