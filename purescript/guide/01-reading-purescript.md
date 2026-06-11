# Part 1 — Reading PureScript

[← Overview](00-overview.md) · [Next: Types and data →](02-types-and-data.md)

The biggest wall for newcomers is that ML-family syntax *looks* alien next to
TypeScript. This part removes the wall. After it, you can parse any line of
basic PureScript, even before you know what every function does.

---

## 1. Function application is a space

No parentheses, no commas. Putting two things next to each other applies the
first to the second:

```purescript
add 3 5            --  add(3, 5)
max 10 x           --  Math.max(10, x)
length "hello"     --  "hello".length
```

```typescript
// TypeScript
add(3, 5);
Math.max(10, x);
"hello".length;
```

Application is **left-associative** and binds **tighter than everything
else**, which explains most "why are there parens there" confusion:

```purescript
add 3 5 + 1        -- (add 3 5) + 1   = 9      — application beats +
add 3 (5 + 1)      -- parens needed to pass 6 as the argument
```

> **Trap:** `f x + 1` is `(f x) + 1`, never `f (x + 1)`. When an argument is
> itself an expression, parenthesize it (or use `$`, §3).

There are no methods — everything is a free function taking its "receiver"
as an argument. (The `.` *does* exist and does what TS dot does — but only
for record fields, [Part 2](02-types-and-data.md). `user.name` is field
access; `length "hello"` is how you call functions.)

---

## 2. Type signatures: arrows and `forall`

A signature is its own line above the definition, joined by `::` ("has
type"):

```purescript
greet :: String -> String
greet name = "Hello, " <> name        -- <> concatenates strings
```

Multi-argument functions separate *arguments* with the same `->` as the
return type:

```purescript
add :: Int -> Int -> Int
add x y = x + y
```

Read: *takes an `Int`, takes an `Int`, returns an `Int`* — last `->` is the
return, everything before is arguments. That covers 95% of signatures.

### Why arrows: currying

The deeper truth: **every function takes exactly one argument.**
`Int -> Int -> Int` is `Int -> (Int -> Int)` — a function returning a
function awaiting the rest. You know this shape; it's the TS arrow-chain:

```typescript
// TypeScript — `add` literally is:
const add = (x: number) => (y: number) => x + y;
const add3 = add(3);    // first argument locked in
add3(10);               // 13
```

```purescript
add 3 5        -- 8 — really (add 3) 5, i.e. add(3)(5)
add3 = add 3   -- partial application: just stop supplying arguments
add3 10        -- 13
```

This is why application is spaces: handing over arguments one at a time is
the normal case, and **partial application is free**. It's used constantly,
along with **underscore sections** — anonymous functions where `_` marks the
missing piece:

```purescript
map (add 3) [1, 2, 3]       -- [4,5,6]   partial application as the mapper
map (_ * 2) [1, 2, 3]       -- [2,4,6]   (_ * 2) is \x -> x * 2
filter (_ > 10) prices      -- TS: prices.filter(p => p > 10)
```

Full lambdas are `\x -> x + 1` (the `\` is an ASCII λ) — exactly
`(x) => x + 1` — but idiomatic code prefers a partial application or a
section when one fits.

⚡ *For Haskell readers: sections always use `_` — `(* 2)` doesn't parse,
`(_ * 2)` does. The underscore generalizes: `_.name` projects a field,
`maximum _.score` style lambdas abound.*

### `forall` — generics, declared explicitly

```purescript
length :: forall a. Array a -> Int
```

A lowercase name in a signature is a **type variable** (a generic), and
PureScript requires you to introduce it with `forall` — exactly TS's `<A>`
declaration site:

```typescript
// TypeScript
function length<A>(xs: A[]): number;
```

Read `forall a.` as `<A>`. More on what type variables buy you in
[Part 2](02-types-and-data.md).

⚡ *Haskell infers the `forall`; PureScript spells it out. Same meaning.*

---

## 3. `$`, `#`, and composition: the plumbing operators

**`$` kills trailing parentheses** — everything to its right is evaluated
first, then passed to the left:

```purescript
show (sum (filter (_ > 0) xs))     -- paren pile-up
show $ sum $ filter (_ > 0) xs     -- same thing
```

When you see `f $ x`, read `f (x)`.

**`#` is `$` flipped — the pipe.** The value flows left to right through the
functions, which is exactly your method-chain reading order:

```purescript
xs # filter (_ > 0) # sum # show
```

```typescript
// TypeScript — same order, same shape
xs.filter(x => x > 0).reduce(sum).toString();
```

**`<<<` and `>>>` compose functions** — gluing functions into a new function
without mentioning the argument (no data flows yet; you're building the
pipeline to use later):

```purescript
countPositive = length <<< filter (_ > 0)   -- right-to-left, like nested calls
countPositive = filter (_ > 0) >>> length   -- left-to-right, like a pipe
```

Both define the same function; `<<<` matches "length of (filter of xs)"
nesting, `>>>` matches pipeline order. Use whichever reads better — TS
developers usually find `>>>` and `#` the natural pair.

⚡ *For Haskell readers: `.` is record access here, so composition is `<<<`;
`&` is `#`. `>>>` exists in both languages.*

---

## 4. Defining functions: equations, patterns, guards

A function can be defined by **multiple equations**, tried top to bottom —
first match wins:

```purescript
describe :: Int -> String
describe 0 = "zero"
describe 1 = "one"
describe n = "many"          -- n matches anything and binds it
```

```typescript
// TypeScript — the same dispatch, written as control flow
const describe = (x: number): string =>
  x === 0 ? "zero" : x === 1 ? "one" : "many";
```

Patterns destructure — like TS destructuring, but checked, and able to test
*values and shapes* at the same time:

```purescript
first :: forall a b. Tuple a b -> a
first (Tuple x _) = x          -- _ = "don't care"
```

Pattern matching's full power arrives with sum types in
[Part 2](02-types-and-data.md), where it becomes an exhaustive `switch` the
compiler audits.

### Guards: per-equation conditions

When you need conditions rather than shapes — `|` reads as "when":

```purescript
grade :: Int -> String
grade score
  | score >= 90 = "A"
  | score >= 80 = "B"
  | otherwise   = "F"        -- otherwise is literally just `true`
```

That's the `if (...) return ...; if (...) return ...;` cascade, flattened.

---

## 5. `where` and `let`: naming the pieces

`where` hangs helper definitions *below* their use — headline first, details
after:

```purescript
bmi :: Number -> Number -> String
bmi weight height = describe (weight / (height * height))
  where
  describe value
    | value < 18.5 = "underweight"
    | value < 25.0 = "normal"
    | otherwise    = "overweight"
```

> **Trap:** a `where` binding is visible in the equation's *body*, but **not
> in its guards** — `bmi weight height | value < 18.5 = ...` with `value`
> defined in the `where` is an `Unknown value` error. Route the value through
> a helper's parameter (as above) or a `let`. ⚡ *This is a real divergence
> from Haskell, where `where` does scope over guards.*

`let … in …` introduces names *before* the expression, and is itself an
expression:

```purescript
cylinderArea r h =
  let side = 2.0 * pi * r * h
      top  = pi * r * r
  in side + 2.0 * top
```

Both are `const` bindings, never mutation. (Note `2.0`, not `2` — `Number`
and `Int` are genuinely different types here; [Part 2](02-types-and-data.md).)

---

## 6. Everything is an expression

Mindset shift #1 in action: no statements, so every construct yields a value
and composes anywhere.

`if` always has an `else` and produces a value — it's the ternary, not the
if-statement:

```purescript
status = if score >= 60 then "pass" else "fail"
```

```typescript
// TypeScript
const status = score >= 60 ? "pass" : "fail";
```

`case` is the expression form of pattern matching (multiple equations are
sugar over it), and `case _ of` is the built-in shorthand for a function
that immediately matches its argument:

```purescript
describe n = case n of
  0 -> "zero"
  1 -> "one"
  _ -> "many"

describe' = case _ of          -- same function — no parameter named at all
  0 -> "zero"
  1 -> "one"
  _ -> "many"
```

No statements means no early `return` — a function's body is one expression
and "returning" is being its value. Where you'd early-return, use a guard or
a `case` branch.

---

## 7. The layout rule (indentation)

Whitespace is significant, but the rule is **alignment**, not nesting depth:

> Continuation lines must be indented **further** than the line that started
> the expression. Things at the **same column** are siblings (next equation,
> next `case` branch, next `where` binding).

```purescript
total = sum
  [ price * qty          -- continuation: further right than `total`
  , shipping
  ]

grade score
  | score >= 90 = "A"    -- guards aligned = siblings
  | otherwise   = "F"
```

If the compiler reports an unexpected token where the code looks fine, you
almost certainly dedented a continuation back to its parent's column —
indent it one step further and it parses.

---

## 8. Names and comments

- `-- line comment`, `{- block comment -}`.
- Capitalization is **enforced**: types, constructors, and modules are
  `PascalCase`; functions and variables `camelCase`. `Foo` vs `foo` tells
  you type-world vs value-world at a glance.
- A trailing `'` is legal in names (`describe'` above): read "a variant of."
- Operators are ordinary functions with symbol names. Any function can go
  infix with backticks (`` 7 `div` 2 ``), any operator prefix with parens
  (`(+) 3 5`).

---

## 9. Bindings, not variables

`x = 5` *defines*, it doesn't *assign* — there is no way for `x` to become
`6` later. The TS model: `const` is the only keyword and every value is
deeply frozen, with the compiler rather than `Object.freeze` doing the
enforcing.

What looks like an update is always a **new value**:

```purescript
user { age = user.age + 1 }    -- a NEW record; `user` is untouched (Part 2)
```

The consequences land immediately: no `for`/`while`, no `i++`, no `push` —
iteration is `map`/`filter`/folds ([Part 3](03-functions-arrays-purity.md)),
accumulation builds new values. The payoff: any expression can be understood
in isolation, because nothing mutates underneath it. (And when you genuinely
need a mutable cell at the edges of a program, that's an *effect* — `Ref`,
[Part 5](05-effect-and-aff.md) — visible in the types like every other
effect.)

---

## One-sentence mental model

PureScript source is one big nested expression: functions apply by
juxtaposition and curry by default, `$`/`#` and `<<<`/`>>>` keep pipelines
readable in either direction, equations/guards/`case` choose branches by
shape, `where` names the pieces — and every name is a `const`, forever.

---

[← Overview](00-overview.md) · [Next: Types and data →](02-types-and-data.md)
