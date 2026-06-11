# Part 3 — Functions, lists, and laziness

[← Types and data](02-types-and-data.md) · [Next: Functor → Applicative → Monad →](04-functor-applicative-monad.md)

Mindset shift #2 said loops become recursion and updates become new values.
This part shows what you actually write instead — which is mostly *not*
recursion, but `map`/`filter`/folds over lists. Then the two practical topics
every beginner hits: lazy evaluation, and why there are three string types.

---

## 1. Lists

The workhorse collection. Syntax you'll recognize, plus ranges:

```haskell
nums  = [1, 2, 3]
λ> [1 .. 5]                 -- [1,2,3,4,5]
λ> [0, 2 .. 10]             -- [0,2,4,6,8,10]  (first two elements set the step)
λ> ['a' .. 'e']             -- "abcde"  (String IS [Char] — see §6)
```

Under the sugar, a list is a sum type you already know how to read after
Part 2 — either empty, or one element **consed** onto a rest:

```haskell
-- conceptually:  data [a] = [] | a : [a]
λ> 1 : [2, 3]               -- [1,2,3]      (: is "cons": prepend, O(1))
λ> [1, 2, 3] ++ [4, 5]      -- [1,2,3,4,5]  (++ appends, O(length of left))
```

Which means lists pattern-match like any other sum type — this pair of
patterns is the idiom of the language:

```haskell
summary :: [String] -> String
summary []       = "no items"
summary (x : xs) = x <> " and " <> show (length xs) <> " more"
```

`(x : xs)` binds the first element and the rest — Python's
`first, *rest = items`, as a checked pattern. The `x`/`xs` naming ("x and
x-es") is universal convention.

> **Trap:** Haskell lists are singly-linked lists, not arrays. Prepend is
> O(1); indexing (`xs !! n`) and append are O(n). Idiomatic code processes
> lists front-to-back as a stream and rarely indexes. (Real array/vector
> types exist in libraries when you need them.)

---

## 2. `map`, `filter`, folds

You already write JS pipelines; the functions have near-identical meanings,
they're just free functions (and partial application from Part 1 replaces the
arrow-function noise):

```haskell
λ> map (* 2) [1, 2, 3]              -- [2,4,6]
λ> filter even [1 .. 10]            -- [2,4,6,8,10]
λ> sum (map (* 2) (filter even [1 .. 10]))   -- 60
```

```typescript
// TypeScript
[1, 2, 3].map(x => x * 2);
range(1, 10).filter(isEven);
range(1, 10).filter(isEven).map(x => x * 2).reduce((a, x) => a + x, 0);
```

`reduce` is called a **fold**, and there are two that matter:

```haskell
foldr f z [a, b, c]  =  a `f` (b `f` (c `f` z))     -- folds from the Right
foldl' f z [a, b, c] =  ((z `f` a) `f` b) `f` c     -- folds from the Left, strictly
```

```haskell
λ> foldl' (+) 0 [1 .. 100]          -- 5050   (import Data.List (foldl'))
λ> foldr (\x acc -> x * 2 : acc) [] [1, 2, 3]   -- [2,4,6] — map, built from foldr
```

JS `reduce` corresponds to `foldl'`: accumulator on the left, eats the list
front to back. Practical guidance, with the *why* deferred to §5:

> **Trap:** for summarizing a list into one strict value (sum, count, max),
> use **`foldl'`** (note the prime — "strict variant", as promised in Part 1).
> Plain `foldl` is a space-leak trap that essentially exists for historical
> reasons; `foldr` is for rebuilding list-like structures and for lazy
> short-circuiting.

In real code you'll reach for the specialized ones before raw folds: `sum`,
`product`, `maximum`, `length`, `and`, `or`, `any`, `all`, `concat`,
`zip`/`zipWith`, `takeWhile`/`dropWhile` — almost all of them folds under the
hood.

---

## 3. List comprehensions

Python's comprehensions, almost symbol for symbol — `|` reads as Python's
`for`, generators left to right, conditions are bare boolean expressions:

```haskell
λ> [x * 2 | x <- [1 .. 10], even x]
[4,8,12,16,20]

pairs = [(x, y) | x <- [1, 2, 3], y <- "ab"]
-- [(1,'a'),(1,'b'),(2,'a'),(2,'b'),(3,'a'),(3,'b')]
```

```python
# Python
[x * 2 for x in range(1, 11) if x % 2 == 0]
[(x, y) for x in [1, 2, 3] for y in "ab"]
```

Comprehensions and `map`/`filter` chains are interchangeable; Haskellers use
comprehensions mostly when combining *multiple* generators (the `pairs`
example — nested loops flattened into one expression).

---

## 4. When you do write recursion

`map`/`filter`/folds cover most iteration. Recursion is the fallback for
everything else — and structurally it's the same two cases as the list type
itself: handle empty, handle `x : xs` using the answer for `xs`.

```haskell
-- Go's accumulating for-loop…
-- sum := 0
-- for _, x := range xs { sum += x }

-- …its direct recursive translation:
sumList :: [Int] -> Int
sumList []       = 0              -- loop didn't run: return the accumulator's start
sumList (x : xs) = x + sumList xs -- one iteration, then "continue" on the rest
```

A `while` loop — loop variables become function arguments, the condition
becomes pattern/guard, the body becomes the recursive call's new arguments:

```python
# Python
def countdown(n):
    while n > 0:
        print(n)        # (printing aside — effects come in Part 5)
        n -= 1
```

```haskell
countdown :: Int -> [Int]
countdown n
  | n <= 0    = []
  | otherwise = n : countdown (n - 1)   -- "n -= 1" is just the argument of the next call
```

When a recursion needs extra state, the idiom is a local helper named `go`
carrying the state as arguments — you'll see `go` in every codebase:

```haskell
average :: [Double] -> Double
average = go 0 0
  where
    go total count []       = total / fromIntegral count
    go total count (x : xs) = go (total + x) (count + 1) xs
```

That's a loop with two mutable variables, made honest: each "iteration" is a
call with the updated values. (No stack overflow worries for this shape —
tail calls don't grow the stack, and §5's laziness changes the cost model
anyway. The actual beginner risk is the `foldl` trap above, not recursion
depth.)

---

## 5. Laziness

Haskell's most distinctive runtime behavior: **nothing is computed until
something demands the result.** Every expression starts life as a **thunk** —
a suspended computation — and is forced only when (and if) it's needed.

You've used this model: Python generators. The difference is opt-in vs
default:

```python
# Python — lazy ONLY because we used generators on purpose
nums = itertools.count(1)              # infinite, fine
evens = (x for x in nums if x % 2 == 0)
list(itertools.islice(evens, 5))       # [2, 4, 6, 8, 10]
```

```haskell
-- Haskell — the same shape is just... lists
λ> take 5 (filter even [1 ..])         -- [2,4,6,8,10]
```

`[1 ..]` is genuinely infinite and that's fine: `take 5` demands five
elements, so five (well, ten) are ever produced. Consequences:

- **Infinite structures are ordinary values.** `cycle [1,2,3]`,
  `repeat 'x'`, `fibs = 0 : 1 : zipWith (+) fibs (tail fibs)` — define the
  whole thing, consume what you need.
- **You can build your own short-circuiting.** `&&`, `||`, and even `if` are
  ordinary functions/sugar, not special forms — safe because arguments
  aren't evaluated until used. Your own functions get this for free:
  `firstJust` over a list of expensive lookups stops at the first hit
  without any explicit early-exit machinery.
- **Pipelines don't build intermediate lists the way you'd fear.**
  `sum (map (*2) (filter even xs))` runs as one pass: each element is
  produced, transformed, and consumed before the next exists. You get
  generator-pipeline efficiency with list-literal syntax.

### When laziness bites

The flip side — at awareness level, since the fix is a habit, not a project:

A thunk that's *stored* instead of *forced* piles up. The classic is plain
`foldl (+) 0 [1..10^7]`: the accumulator never gets demanded during the loop,
so it becomes the unevaluated tower `(((0+1)+2)+3)+…` — millions of thunks
held in memory (a **space leak**), all collapsing only at the end.
**`foldl'`** forces the accumulator at each step, keeping it a plain number.
That's the entire §2 trap, explained.

The habit: when *accumulating* a value, use the strict variant — `foldl'`,
strict `Data.Map.Strict`, strict record fields (`!Int`, [Part 6](06-structuring-programs.md)).
Streaming and transforming, laziness is your friend; accumulating, force it.

---

## 6. Strings: `String`, `Text`, and `ByteString`

The classic real-world gotcha, best learned before it's confusing. There are
three "stringish" types:

| Type | What it is | Use it for |
| ---- | ---------- | ---------- |
| `String` | literally `[Char]` — a linked list of characters | the default in tutorials, the Prelude, `Show`; fine for small things |
| `Text` (`Data.Text`) | packed UTF-16 text | **actual text in real programs** — the default in app code and serious libraries |
| `ByteString` | packed raw bytes | binary data, network I/O, file contents before decoding |

`String` being a linked list is elegant (every list function works on it —
that's why `['a'..'e']` was a string) and inefficient (per-character boxes
and pointers). Real code uses `Text`; mentally file `String ≈ legacy default,
Text ≈ the real one, ByteString ≈ Buffer/[]byte`.

The friction: string *literals* are `String` by default, so passing
`"hello"` where `Text` is expected won't typecheck. The universal fix is the
`OverloadedStrings` extension, which makes literals adapt to the expected
type — practically every real project enables it:

```haskell
{-# LANGUAGE OverloadedStrings #-}     -- file-level pragma, first line of the file
import Data.Text (Text)
import qualified Data.Text as T        -- the import idiom is explained in Part 6

shout :: Text -> Text
shout t = T.toUpper t <> "!"           -- "!" is Text here, because that's what's expected
```

(How a literal can be polymorphic: a typeclass dispatching on the expected
type — the same return-type dispatch from
[Part 2 §6](02-types-and-data.md#6-typeclasses-just-enough-to-read-deriving).
Numeric literals work the same way, which is why `5` can be an `Int` or a
`Double`.)

> **Trap:** if GHC complains
> `Couldn't match type ‘[Char]’ with ‘Text’` — that's this. A `String`
> (`[Char]`) met a `Text`. Enable `OverloadedStrings` for literals, or
> convert explicitly with `T.pack :: String -> Text` / `T.unpack` the other way.

---

## One-sentence mental model

A list is `[] | x : xs` and everything follows: `map`/`filter`/`foldl'`
replace loops, comprehensions are Python's, recursion with a `go` helper is
the honest while-loop — all of it lazily produced on demand, so infinite is
fine and pipelines fuse, with the one rule *force what you accumulate* — and
the day a string won't typecheck, the answer is `Text` + `OverloadedStrings`.

---

[← Types and data](02-types-and-data.md) · [Next: Functor → Applicative → Monad →](04-functor-applicative-monad.md)
