# Part 1 — Reading Haskell

[← Overview](00-overview.md) · [Next: Types and data →](02-types-and-data.md)

The biggest wall for newcomers isn't monads — it's that Haskell *looks* alien.
This part removes that wall. After it, you can parse any line of basic Haskell,
even if you don't yet know what every function does.

---

## 1. Function application is a space

No parentheses, no commas. Putting two things next to each other applies the
first to the second.

```haskell
add 3 5          --  add(3, 5)
max 10 x         --  Math.max(10, x)
length "hello"   --  "hello".length
```

```typescript
// TypeScript
add(3, 5);
Math.max(10, x);
"hello".length;
```

Application is **left-associative** and binds **tighter than everything else**,
which explains most "why are there parens there" confusion:

```haskell
add 3 5 + 1        -- (add 3 5) + 1   = 9      — application beats +
add 3 (5 + 1)      -- need parens to pass 6 as the argument
negate (length xs) -- without parens: (negate length) xs — applies negate to a function. Type error.
```

> **Trap:** `f x + 1` is `(f x) + 1`, never `f (x + 1)`. When an argument is
> itself an expression, parenthesize it (or use `$`, §3).

There are no methods and no `obj.method()` — everything is a free function
that takes its "receiver" as an argument. `length "hello"`, not
`"hello".length()`.

---

## 2. Type signatures: reading the arrows

A signature is a separate line above the definition, connected by `::`
(read: "has type").

```haskell
greet :: String -> String
greet name = "Hello, " <> name
```

Multi-argument functions look strange at first — the arguments are separated
by `->`, the same arrow as the return type:

```haskell
add :: Int -> Int -> Int
add x y = x + y
```

Read it as: *takes an `Int`, takes an `Int`, returns an `Int`* — everything
after the last `->` is the return type, everything before is arguments. That
reading is all you need 95% of the time.

### Why arrows everywhere: currying

The deeper truth: **every Haskell function takes exactly one argument.**
`Int -> Int -> Int` associates right — it's really `Int -> (Int -> Int)`:
a function that takes an `Int` and returns *another function* waiting for the
second `Int`. You already know this shape from TypeScript:

```typescript
// TypeScript — `add` literally is this arrow-chain:
const add = (x: number) => (y: number) => x + y;
add(3)(5);    // 8
const add3 = add(3);   // a new function, first arg locked in
add3(10);     // 13
```

```haskell
add 3 5        -- 8        (really (add 3) 5 — same as add(3)(5))
add3 = add 3   -- partial application: no special syntax, just stop early
add3 10        -- 13
```

This is why application uses spaces, not `f(x, y)`: supplying arguments one at
a time is the normal case, and **partial application is free**. It's used
constantly:

```haskell
map (add 3) [1, 2, 3]      -- [4,5,6]
filter (> 10) prices        -- sections: (> 10) is \x -> x > 10
map (* 2) xs                -- (* 2) is \x -> x * 2
```

Lambdas exist too — `\x -> x + 1` (the `\` is meant to look like λ) — the
direct equivalent of `(x) => x + 1`. But idiomatic Haskell often partially
applies a named function instead of writing a lambda.

### Lowercase types are type variables

```haskell
length :: [a] -> Int
```

A lowercase name in a signature (`a`) is a **type variable** — implicit
generics. No `<T>` declaration needed; lowercase *is* the declaration.

```typescript
// TypeScript
function length<A>(xs: A[]): number;
```

More on these in [Part 2](02-types-and-data.md).

---

## 3. `$` and `.` — the two operators you'll see most

**`$` is "apply, but with the lowest precedence."** Its only job is killing
trailing parentheses. Everything to its right is evaluated first, then passed
to what's on its left:

```haskell
print (sum (filter even [1..10]))   -- paren pile-up
print $ sum $ filter even [1..10]   -- same thing
```

When you see `f $ x`, read it as `f (x)`.

**`.` is function composition** — it glues functions into pipelines, like
nesting calls without naming the intermediate value:

```haskell
(f . g) x  =  f (g x)         -- the definition

countEvens :: [Int] -> Int
countEvens = length . filter even   -- "filter evens, then take the length"
```

```typescript
// TypeScript — same pipeline, written inside-out or with intermediates
const countEvens = (xs: number[]) => xs.filter(isEven).length;
```

> **Reading order:** `.` chains read **right to left** — `length . filter even`
> runs `filter even` first. Like nested calls `length(filter(even, xs))`, data
> flows right→left. (If you've used Unix pipes: same idea, opposite direction.)

Defining `countEvens` without mentioning its argument
(`countEvens = length . filter even` rather than
`countEvens xs = length (filter even xs)`) is called **point-free style**.
Fine in small doses; don't force it.

---

## 4. Defining functions: equations, patterns, guards

A function can be defined by **multiple equations**, tried top to bottom — the
first whose patterns match wins. This is pattern matching in its natural
habitat:

```haskell
describe :: Int -> String
describe 0 = "zero"
describe 1 = "one"
describe n = "many"        -- n matches anything and binds it
```

```python
# Python — the same shape with structural pattern matching
def describe(x: int) -> str:
    match x:
        case 0: return "zero"
        case 1: return "one"
        case n: return "many"
```

Patterns can destructure, like TS destructuring but checked and exhaustive:

```haskell
first :: (a, b) -> a
first (x, _) = x            -- _ = "I don't care about this part"

startsWith :: String -> String
startsWith ""      = "empty"
startsWith (c:_)   = "starts with " <> [c]   -- (c:rest) splits head/tail
```

Pattern matching gets its full treatment with sum types in
[Part 2](02-types-and-data.md) — matching on *which variant* a value is, where
it becomes Haskell's exhaustive `switch`.

### Guards: boolean conditions per equation

When you need conditions rather than shapes, use guards — `|` reads as "when":

```haskell
grade :: Int -> String
grade score
  | score >= 90 = "A"
  | score >= 80 = "B"
  | otherwise   = "F"       -- otherwise is literally just True
```

This replaces the `if (...) return ...; if (...) return ...;` cascade. Guards
are tried top to bottom; `otherwise` is the catch-all.

---

## 5. `where` and `let`: naming the pieces

`where` hangs helper definitions *below* the expression that uses them —
read the headline first, the details after:

```haskell
bmi :: Double -> Double -> String
bmi weight height
  | value < 18.5 = "underweight"
  | value < 25.0 = "normal"
  | otherwise    = "overweight"
  where
    value = weight / (height * height)
```

`let … in …` is the same idea but introduces names *before* the expression,
and is itself an expression:

```haskell
cylinderArea r h =
  let side = 2 * pi * r * h
      top  = pi * r * r
  in  side + 2 * top
```

Rule of thumb: `where` for "here's the formula, definitions below" (the common
style); `let` when you want the bindings inline mid-expression (and inside
`do` blocks, [Part 5](05-io-and-effects.md)). Both are just local `const`
bindings — never mutation.

---

## 6. Everything is an expression

Mindset shift #1 in action. Haskell has no statements — every construct
produces a value, so every construct can sit anywhere a value can.

`if` always has an `else` and yields a value — it's TS's ternary, not its
`if` statement:

```haskell
status = if score >= 60 then "pass" else "fail"
```

```typescript
// TypeScript — the ternary IS the right analogy, the if-statement isn't
const status = score >= 60 ? "pass" : "fail";
```

`case` is the expression form of pattern matching (equations from §4 are
sugar over it):

```haskell
describe n = case n of
  0 -> "zero"
  1 -> "one"
  _ -> "many"
```

Because there are no statements, there is no early `return` — a function's
body is one expression, and "returning" is just being the value of it. Where
you'd reach for an early return, Haskell uses a guard or a `case` branch.

> **Trap for Python folks:** there's no statement/expression divide to work
> around — no equivalent of "can't use `if` inside a lambda." Every construct
> composes everywhere.

---

## 7. The layout rule (indentation)

Like Python, whitespace is significant; unlike Python, the rule is about
**alignment**, not nesting level:

> Continuation lines of an expression must be indented **further** than the
> line that started it. Things aligned at the **same column** are siblings
> (new equations, new `case` branches, new `where` bindings).

```haskell
total = sum [ price * qty            -- continuation lines: indented further
            , shipping
            ]

grade score
  | score >= 90 = "A"                -- guards aligned = siblings
  | otherwise   = "F"
```

If GHC says `parse error (possibly incorrect indentation)`, you almost
certainly dedented a continuation line back to (or past) its parent's column.
Indent it further right and it parses.

---

## 8. Names and comments

- `-- line comment` and `{- block comment -}`.
- Capitalization is **enforced, not convention**: types, constructors, and
  modules are `PascalCase`; functions and variables are `camelCase`. Seeing
  `Foo` vs `foo` tells you which world you're in ([Part 2](02-types-and-data.md)
  leans on this).
- A trailing `'` (prime) is legal in names: `foldl'`, `go'`. Read it as "a
  variant of" — often a stricter or modified version.
- Operators are ordinary functions with symbol names. Any function can be
  used infix with backticks (`` 7 `div` 2 ``), and any operator used prefix
  with parens (`(+) 3 5`).

---

## 9. Bindings, not variables

`x = 5` doesn't *assign* — it *defines*. There is no way to make `x` be `6`
later. The mental model from TypeScript: `const` is the only keyword, and
every value is deeply frozen.

```haskell
x = 5
x = 6        -- error: "Multiple declarations of ‘x’" — not reassignment, a name clash
```

What looks like update is always **a new binding** (often shadowing in a
nested scope) or **a new value**:

```haskell
addYear user = user { age = age user + 1 }   -- a NEW record; old one untouched
```

Consequences you'll feel immediately:

- No `for`/`while` — iteration is recursion or `map`/`filter`/`fold`
  ([Part 3](03-functions-lists-laziness.md)).
- No `i++`, no accumulating into a mutable array — you build new values.
- The payoff: any expression can be understood in isolation. Nothing mutates
  underneath you, so "what is `x`?" always has one answer.

---

## One-sentence mental model

Haskell source is **one big nested expression**: functions apply by
juxtaposition and curry by default, `$` and `.` keep the nesting readable,
equations/guards/`case` choose between branches by shape, `where` names the
pieces — and every name is a `const`, forever.

---

[← Overview](00-overview.md) · [Next: Types and data →](02-types-and-data.md)
