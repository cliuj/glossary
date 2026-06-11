# Part 5 — IO: how a pure language does side effects

[← Functor → Applicative → Monad](04-functor-applicative-monad.md) · [Next: Structuring programs →](06-structuring-programs.md)

Mindset shift #4, finally in full. Everything so far has been *pure*: same
inputs, same output, nothing else happens. But programs must print, read
files, and talk to networks. This part shows the trick Haskell uses to allow
all of that without giving up purity — and why, after Part 4, you already
know 90% of it: **`IO` is just another monad**, and do-notation is how you
script it.

---

## 1. The trick: actions are values

A pure function cannot print — printing is an effect, and effects would make
`f x` mean different things at different times. Haskell's resolution:

> A value of type `IO a` is a **description of an action** that, *when
> executed by the runtime*, performs side effects and produces an `a`.
> Pure code can build, combine, and pass around these descriptions freely —
> describing an action has no side effects. **Execution** happens in exactly
> one place: whatever `main` evaluates to is the action the runtime runs.

The TypeScript intuition is a Promise *factory* — note the thunk:

```typescript
// TypeScript
const greet: () => Promise<void> = () => writeLine("hi");
// Holding `greet` prints nothing. Composing it into a bigger pipeline
// prints nothing. Only the runtime invoking it prints.
```

```haskell
greet :: IO ()          -- a description of "print hi". Holding it prints nothing.
greet = putStrLn "hi"

main :: IO ()           -- main IS the program: the one description that gets run
main = greet
```

So `putStrLn "hi"` doesn't print — it *is* a printing-action, a first-class
value you can put in a list, pass to a function, or use three times. The
type makes effects visible everywhere: `Int -> Int` *cannot* touch a file or
launch missiles; `Int -> IO Int` declares it might. In Go/TS you find out
what a function touches by reading its body (or its docs, optimistically);
here the signature is a guarantee the compiler enforces.

`()` — "unit" — is the empty tuple, standing in for "no interesting result":
`IO ()` is an action run only for its effect, Haskell's `Promise<void>`.

---

## 2. Writing programs: do-notation over IO

All of Part 4 transfers wholesale. `IO`'s `>>=` means "run this action, then
feed its result to the next" — sequencing of effects — so a do-block over IO
reads exactly like the imperative code it replaces:

```haskell
main :: IO ()
main = do
  putStrLn "What's your name?"
  name <- getLine                          -- run the action, bind its result
  let greeting = "Hello, " <> name <> "!"  -- pure computation: no action run
  putStrLn greeting
```

```typescript
// TypeScript — the async/await analogy from Part 4, now load-bearing
async function main(): Promise<void> {
  console.log("What's your name?");
  const name = await readLine();
  const greeting = `Hello, ${name}!`;
  console.log(greeting);
}
```

The `<-` vs `let` distinction from Part 4's trap, now in its natural
habitat: `<-` *runs* an `IO a` and binds the `a`; `let` names a pure value.
`getLine` has type `IO String` — a `String` is only on the other side of a
`<-`.

> **Trap:** the first error every beginner hits is using an `IO String`
> where a `String` is wanted — e.g. `putStrLn (getLine)`. GHC says
> `Couldn't match type ‘IO String’ with ‘[Char]’`. The fix is always the
> same: bind it first (`s <- getLine`), *then* use `s`. There is no
> `unwrap`/`await`-inline; the bind is the unwrap.

The everyday vocabulary:

```haskell
putStrLn  :: String -> IO ()             -- console.log
putStr    :: String -> IO ()             -- ...without the newline
print     :: Show a => a -> IO ()        -- putStrLn . show — for values, not strings
getLine   :: IO String
readFile  :: FilePath -> IO String
writeFile :: FilePath -> String -> IO ()
getArgs   :: IO [String]                 -- from System.Environment — os.Args[1:]
```

(`print` vs `putStrLn`: `print "hi"` shows the quotes — it's `show`-ing the
string. `putStrLn` for text meant for humans, `print` for debugging values.
For `Text` from Part 3, the equivalents live in `Data.Text.IO`.)

And the Part 4 combinators come alive — an action is a value, so a list of
actions is an ordinary list, and traversal runs them in order:

```haskell
main :: IO ()
main = do
  args  <- getArgs
  files <- traverse readFile args      -- read every file: [String] of contents
  let total = sum (map (length . lines) files)
  when (total > 1000) $ putStrLn "that's a lot of lines"
  mapM_ putStrLn args                  -- effect per element, results discarded
  print total
```

---

## 3. The boundary is the architecture

The deeper consequence of `IO` showing up in types: **the pure/impure split
becomes a design line the compiler patrols.** You already aim for this in Go
and TS — handlers thin, business logic in plain testable functions, effects
pushed to the edges — "functional core, imperative shell." The difference is
enforcement: in TS, nothing stops a "pure" helper from quietly calling
`fetch`. In Haskell, that helper would need `IO` in its type, every caller
would see it, and the type checker makes drift impossible.

The idiomatic program shape:

```haskell
-- THE CORE: pure decisions — trivially testable, no mocks, no setup
data Report = Report { lineCount :: Int, todoCount :: Int }

analyze :: String -> Report
render  :: Report -> String

-- THE SHELL: a thin main that ferries data in and out
main :: IO ()
main = do
  [path] <- getArgs
  text   <- readFile path
  putStrLn (render (analyze text))      -- core sandwiched between two effects
```

Working heuristic: when a function wants `IO` just to *get at* a value, flip
it — do the I/O in the caller, pass the value in. `analyze :: String -> Report`,
not `analyze :: FilePath -> IO Report`. Keep `IO` at the rim; let the core
stay pure. ([Part 7](07-transformers-and-readert.md) is about what to do when
the shell itself grows config, logging, and state — the rim gets structure of
its own.)

---

## 4. Errors in IO: exceptions exist after all

Surprise: alongside `Maybe`/`Either`, Haskell has real runtime exceptions —
`readFile` on a missing file *throws*. The Go panic/error split is the right
map:

| Kind | Haskell | Go analogue | Use for |
| ---- | ------- | ----------- | ------- |
| Expected, recoverable, caller must handle | `Maybe` / `Either` in the return type | `(val, err)` | parse failures, lookups, validation — *domain* errors |
| Exceptional, environmental | exceptions in `IO` (`throwIO`, `catch`) | `panic` / I/O errors | missing files, dead sockets, out of memory |

The working rules:

- **Pure functions: failure goes in the return type.** A pure function that
  can fail says so — `parse :: String -> Either Err Order`. (A few Prelude
  functions break this rule — `head []` throws. Modern style avoids these
  *partial functions*; prefer pattern matching or total variants.)
- **IO code: exceptions are in play whether you like it or not**, so handle
  them at the boundary where you can respond — same place you'd check
  `err != nil` on an `os.Open`:

```haskell
import Control.Exception (try, IOException)

main :: IO ()
main = do
  result <- try (readFile "config.txt") :: IO (Either IOException String)
  case result of
    Left  _    -> putStrLn "no config, using defaults"
    Right text -> putStrLn ("config: " <> text)
```

`try` converts a throwing action into the `Either` you already know how to
chain; `catch`/`handle` are the handler-style alternatives. The pattern to
internalize: **catch at the rim, convert to `Either`/`Maybe`, let the pure
core never know exceptions exist.**

---

## 5. Printf debugging in a pure language

"How do I print from a pure function while debugging?" — `Debug.Trace`, the
sanctioned cheat:

```haskell
import Debug.Trace (trace, traceShow)

area :: Shape -> Double
area (Circle r) = trace ("circle with r = " <> show r) (pi * r * r)
--                ^ prints to stderr when evaluated, returns the 2nd argument
```

`trace` is a pure-looking function that prints — deliberately breaking the
rules, visibly, temporarily. Two caveats: laziness means it prints when the
value is *demanded* (which may be never, or once despite three calls —
Part 3 §5 in action), and it's for sessions, not commits. For real
diagnostics, return the information or log in `IO`.

---

## One-sentence mental model

`IO a` is a *description* of an effectful step — building one is pure,
`main` is the one that runs, do-notation chains them like async/await — so
the signature tells you what a function can touch, the pure core stays free
of effects and exceptions (catch at the rim, convert to `Either`), and
`trace` is the acknowledged cheat code for debugging.

---

[← Functor → Applicative → Monad](04-functor-applicative-monad.md) · [Next: Structuring programs →](06-structuring-programs.md)
