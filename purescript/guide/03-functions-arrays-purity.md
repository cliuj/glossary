# Part 3 — Functions, arrays, and purity

[← Types and data](02-types-and-data.md) · [Next: Functor → Applicative → Monad →](04-functor-applicative-monad.md)

Mindset shift #2 said loops become `map`/`filter`/folds and updates become
new values. This part shows what you actually write — over `Array`, which is
a real JS array underneath — plus the recursion rules of a strict language
that compiles to JS, and the string toolbox.

---

## 1. Arrays: JS arrays, immutably used

`Array a` is *the* PureScript collection — literally a JS array at runtime,
with an API that never mutates (every operation returns a new array):

```purescript
import Data.Array ((..), (!!), uncons, cons, snoc)

nums = [1, 2, 3]                 -- a JS array in the output, verbatim
> 1 .. 5                         -- ranges: the .. operator from Data.Array
[1,2,3,4,5]
> [1, 2] <> [3]                  -- concat is <>, the universal combine
[1,2,3]
> cons 0 [1, 2]                  -- prepend → new array
[0,1,2]
```

Two API choices reflect the no-`null`, no-exceptions worldview — operations
that can miss return `Maybe`:

```purescript
> [10, 20, 30] !! 1              -- indexing is SAFE
Just 20
> [10, 20, 30] !! 9
Nothing                          -- not undefined, not a throw
> head []                        -- Data.Array.head :: Array a -> Maybe a
Nothing
```

```typescript
// TypeScript: xs[9] is number | undefined only if you enabled
// noUncheckedIndexedAccess — and even then ! turns it back off.
```

Array *patterns* match fixed lengths only — there's no rest/spread pattern;
"first and rest" is a function, `uncons`:

```purescript
describe :: Array Int -> String
describe []     = "empty"
describe [x]    = "just " <> show x
describe [x, y] = "a pair summing to " <> show (x + y)
describe xs     = "many (" <> show (length xs) <> " items)"

case uncons xs of                          -- Python's  first, *rest = xs
  Just { head: h, tail: t } -> ...         -- note: returns a record, in a Maybe
  Nothing                   -> ...
```

> **Trap:** `cons`/`uncons` on arrays copy — they're O(n), not O(1). Arrays
> are for indexing, bulk transforms, and interop; if an algorithm wants
> cheap head/tail recursion, that's what `Data.List` (a linked list, exactly
> like Haskell's) is for. In UI code you'll use `Array` ~95% of the time.

---

## 2. `map`, `filter`, folds

The JS pipeline vocabulary, as free functions (with Part 1's sections
replacing the lambda noise):

```purescript
import Data.Foldable (foldl, foldr, sum, maximum, any, all)

> map (_ * 2) [1, 2, 3]                  -- [1,2,3].map(x => x * 2)
[2,4,6]
> filter odd (1 .. 10)                   -- .filter(isOdd)
[1,3,5,7,9]
> foldl (+) 0 (1 .. 100)                 -- .reduce((acc, x) => acc + x, 0)
5050
> foldl (\acc s -> acc <> ", " <> s) "start" ["a", "b"]
"start, a, b"
```

`foldl` is JS `reduce`: accumulator on the left, list eaten front to back.
`foldr` folds from the right — its niche is rebuilding structures and you
can meet it as needed.

⚡ *For Haskell readers: strict language, no thunk buildup — plain `foldl`
is the workhorse; there is no `foldl'` and no space-leak lore to import.*

As in TS, you rarely write raw `reduce`: the specialized folds (`sum`,
`maximum`, `any`, `all`, `length`, `elem`, plus `Data.Array`'s `sortBy`,
`groupBy`, `zip`, `zipWith`, `take`, `drop`, `takeWhile`, `mapMaybe`,
`catMaybes`) cover most needs. Two worth singling out because they encode
the `Maybe` discipline:

```purescript
> maximum ([] :: Array Int)             -- no elements → no maximum. Maybe!
Nothing
> mapMaybe Data.Int.fromString ["1", "x", "2"]    -- map + keep the Justs
[1,2]
```

---

## 3. When you do write recursion

Most "loops" are a `map`/`filter`/fold one-liner. For everything else,
recursion — same two-case shape as the data, with loop state as arguments:

```typescript
// TypeScript
function countdown(n: number): number[] {
  const out: number[] = [];
  while (n > 0) { out.push(n); n -= 1; }
  return out;
}
```

```purescript
countdown :: Int -> Array Int
countdown n
  | n <= 0    = []
  | otherwise = cons n (countdown (n - 1))   -- "n -= 1" is the next call's argument
```

When the recursion needs an accumulator, the idiom is a local `go` helper —
a while-loop made honest:

```purescript
average :: Array Number -> Number
average xs = go 0.0 0 xs
  where
  go total count arr = case uncons arr of
    Nothing                   -> total / toNumber count
    Just { head: x, tail: t } -> go (total + x) (count + 1) t
```

### The strict-language fine print: the JS stack is real

Here's where compiling to JS shows through. **Self-recursive tail calls are
optimized into JS loops** — this is a real guarantee, not folklore; compile
`go` above and the output contains a `while`-style `$tco_loop`, so a million
iterations is fine. But the optimization covers *direct self-calls in tail
position only*:

- `countdown` above is **not** tail-recursive (the `cons` happens *after*
  the recursive call returns), so a huge `n` can blow the JS call stack.
- Mutual recursion (`f` calls `g` calls `f`) is not optimized either.

The practical rules, in order: prefer `Data.Array`'s bulk functions (they're
loop-based and fast); when recursing over big inputs, shape it as a
tail-recursive `go` with accumulators; and if an algorithm truly can't be —
that's rare in UI code — `purescript-tailrec` exists. For everyday work,
"use folds, and `go` when you must" is the whole discipline.

---

## 4. Purity and strictness: it runs like the JS you'd write

Two facts that together make PureScript predictable:

**It's strict.** Expressions evaluate where they appear, top to bottom,
exactly like TS. There are no infinite lists, no deferred thunks, no
evaluation-order puzzles; what you read is the order it runs.
(`&&` and `||` still short-circuit — the compiler inlines them to JS's
operators — and `if`/`case`/guards only run the branch taken, so the
control-flow laziness you actually rely on in TS is all there. Opt-in
lazy values exist in `Data.Lazy` for the rare memoize-on-demand case.)

**It's pure.** Same arguments, same result, nothing else happens — for
*every* function in Parts 1–4. Mutation, randomness, time, and I/O are all
quarantined behind `Effect` ([Part 5](05-effect-and-aff.md)). Combined with
strictness, this gives you a powerful refactoring license: any expression
can be extracted to a `where` binding, inlined back, reordered with its
neighbors, or deduplicated — and the program *cannot* change behavior,
because no expression depends on when or how often it runs. (The compiler
exploits the same license: your `map (f <<< g)` pipelines and curried calls
come out as plain, readable JS functions — open `output/` and look, it's
genuinely instructive.)

```typescript
// TypeScript — neither license exists:
const a = items.pop();       // extract/reorder this and behavior changes
const b = cache.get(key) ?? expensive();  // run-twice ≠ run-once
```

---

## 5. Strings: one type, two views

Good news first: `String` is the JS string — one type, UTF-16, with
literals, `<>` concatenation, and `Data.String`'s toolbox
(`toUpper`, `trim`, `split`, `joinWith`, `contains`, `replace`, …):

```purescript
import Data.String (toUpper, split, joinWith, Pattern(..))

> toUpper "hello"
"HELLO"
> split (Pattern ",") "a,b,c"
["a","b","c"]
> joinWith "-" ["x", "y"]
"x-y"
```

(That `Pattern` newtype wrapping the separator is [Part 2 §5](02-types-and-data.md#5-type-vs-newtype)
in the wild — it exists so search-strings can't be confused with
subject-strings at call sites.)

The fine print is the same one TS has, made explicit: a JS string is a
sequence of UTF-16 *code units*, so emoji and other astral characters count
as 2. PureScript surfaces the choice as two modules — `Data.String.CodeUnits`
(fast, JS `.length` semantics) and `Data.String.CodePoints` (correct for
human-perceived characters; the default `Data.String.length` counts code
points). If you never indexed into emoji in TS, you'll never notice; if you
do, the module name says which semantics you're getting.

---

## One-sentence mental model

`Array` is the JS array used immutably — safe indexing returns `Maybe`,
pipelines are `map`/`filter`/`foldl` (plain `foldl`: strict language, no
leak lore) — recursion is a tail-recursive `go` that genuinely compiles to a
JS loop, evaluation is strict and pure so code runs in the order written and
refactors can't change behavior, and strings are JS strings with the
code-unit/code-point choice made visible.

---

[← Types and data](02-types-and-data.md) · [Next: Functor → Applicative → Monad →](04-functor-applicative-monad.md)
