# Part 7 — Monad transformers and the ReaderT pattern

[← Structuring programs](06-structuring-programs.md) · [Back to overview](00-overview.md)

The capstone, and the part that unlocks *reading real application code*.
Open any production Haskell service and the signatures don't say `IO a` —
they say `App a`, `ReaderT Env IO a`, or `(MonadReader Env m, MonadIO m) => m a`.
This part builds up to those, the same way the codebases did: by hitting a
problem `IO` alone doesn't solve.

---

## 1. The pain: every function takes the environment

Part 5's advice was "keep `IO` at the rim." But a real shell has cross-cutting
needs — config, a logger, a DB pool. The direct solution is the one you'd
write in Go: thread a context/deps struct through *every* function:

```haskell
data Env = Env { verbose :: Bool, apiBase :: String }

logMsg    :: Env -> String -> IO ()
fetchUser :: Env -> UserId -> IO User
syncAll   :: Env -> [UserId] -> IO ()
syncAll env ids = do
  logMsg env "starting"                    -- env, env, env...
  users <- traverse (fetchUser env) ids
  logMsg env ("synced " <> show (length users))
```

```go
// Go — the same ritual
func syncAll(deps *Deps, ids []UserID) error {
    logMsg(deps, "starting")
    ...
}
```

This *works* (and small programs should do exactly this). The annoyance is
pure plumbing: `env` is passed everywhere and inspected almost nowhere.
Every mainstream language grows machinery for this — Go embeds deps in a
service struct and uses methods; TS uses DI containers or closure scope.
Haskell's machinery is a monad.

---

## 2. Three single-purpose monads

Part 4's punchline was "the instance decides what `>>=` means." Three
library monads exist precisely to abstract the three kinds of plumbing:

| Monad | `>>=` threads... | Replaces | Key operations |
| ----- | ---------------- | -------- | -------------- |
| `Reader r` | a read-only environment `r` into every step | the `env ->` parameter ritual above | `ask` (get the env) |
| `State s` | a value `s` that steps can read *and replace* | passing accumulator in, returning it out | `get`, `put`, `modify` |
| `Except e` | early exit carrying an `e` | `Either` chains (it ≈ `Either` with better combinators) | `throwError`, `catchError` |

`Reader` is the one to actually internalize (it's where this part ends up).
A `Reader Env a` is a computation that may consult an `Env` it was never
explicitly handed:

```haskell
import Control.Monad.Reader

greetingFor :: String -> Reader Env String
greetingFor name = do
  env <- ask                              -- summon the environment
  pure (if verbose env then "Hello, " <> name <> ", welcome back!"
                       else "Hi, " <> name)

λ> runReader (greetingFor "Ada") (Env True "https://api")
"Hello, Ada, welcome back!"
```

No `Env` parameter in sight, yet `ask` produces it — because
`Reader Env a` is secretly just `Env -> a`, and `>>=` composes the
functions so the same `Env` flows to every step. Closure-scope DI, made
explicit in the type. `runReader` is where the actual value enters: supplied
once, at the edge.

And `State` in one glance — the threading is invisible but the type is
honest (`State s a` ≈ `s -> (a, s)`):

```haskell
nextId :: State Int Int
nextId = do
  n <- get
  put (n + 1)
  pure n

λ> runState (traverse (\name -> (,) name <$> nextId) ["a", "b", "c"]) 100
([("a",100),("b",101),("c",102)],103)
```

---

## 3. The catch — and the fix: transformers

The catch: `greetingFor` lives in `Reader`, your I/O lives in `IO`, and
**you can't use two monads in one do-block.** Monads don't compose
automatically — there is no general way to mush `Reader Env` and `IO` into
one monad. (This is *the* limitation that shapes Haskell application
architecture.)

The fix: each effect ships as a **transformer** — a version with a slot for
*another monad underneath*. `ReaderT r m a` is `Reader` layered on any `m`:

```haskell
Reader r a    ≈   r -> a            -- the plain version: base is "nothing"
ReaderT r m a ≈   r -> m a          -- the transformer: base is m
ReaderT r IO a ≈  r -> IO a         -- ...so this is exactly "an IO action that gets an Env"
```

Now the plumbing problem from §1 dissolves — same code, no `env` arguments,
*and* I/O still available:

```haskell
type App a = ReaderT Env IO a

logMsg :: String -> App ()
logMsg s = do
  env <- ask                              -- Reader powers: the env, on demand
  when (verbose env) $ liftIO (putStrLn s)  -- IO powers: via liftIO

syncAll :: [UserId] -> App ()
syncAll ids = do
  logMsg "starting"
  users <- traverse fetchUser ids         -- no env threading anywhere
  logMsg ("synced " <> show (length users))

main :: IO ()
main = runReaderT (syncAll ids) (Env True "https://api")
--     ^ the env is supplied ONCE, here, and flows to every ask
```

The one new word is **`liftIO`**: inside a transformer stack, base-monad
actions must be *lifted* to the outer layer. `putStrLn s :: IO ()` is not an
`App ()` — `liftIO` converts it. (The general `lift` hoists one layer; `liftIO`
jumps straight to `IO` from any depth. When GHC says
`Couldn't match type ‘IO’ with ‘ReaderT Env IO’`, it wants a `liftIO`.)

### Reading a stack type

Transformers nest, and you read the capabilities off the type from the
outside in, then run/peel them **outside-in** with one `run…` per layer:

```haskell
ReaderT Env (ExceptT AppError IO) a
-- capabilities: ask (Env), throwError (AppError), liftIO (IO)

runExceptT (runReaderT action env) :: IO (Either AppError a)
--          ^ peel ReaderT first (outermost), then ExceptT; IO is what remains
```

That final type is the honest shape of the program: an I/O action producing
either an error or an answer. The stack was never magic — just nested
wrappers, each adding one verb and one `run` function.

---

## 4. mtl style: capabilities as constraints

Real codebases often write the *same functions* with classes instead of a
concrete stack — this is the `mtl` library, and it's mostly a reading skill:

```haskell
logMsg :: (MonadReader Env m, MonadIO m) => String -> m ()
logMsg s = do
  env <- ask
  when (verbose env) $ liftIO (putStrLn s)
```

Translate `(MonadReader Env m, MonadIO m) => m ()` as: "runs in *any* monad
`m` that can supply an `Env` and perform I/O." It's the same move as Go's
interface-typed parameters — depend on the capability, not the concrete
type — applied to the monad itself. Benefits: functions don't change when
the app's stack does, tests can swap in a different `m`, and a signature's
constraints are a precise list of what the function can do (a `logMsg`
without `MonadState` *cannot* touch state). When you see `ask`, `throwError`,
or `liftIO` with no stack in the signature, mtl classes are why it works.

---

## 5. The ReaderT pattern: how real apps are built

Production codebases overwhelmingly converge on one specific stack — worth
knowing as a *named architecture*, because you'll see it everywhere:

```haskell
data Env = Env                            -- ALL the app's dependencies, one record
  { config  :: Config
  , logger  :: Text -> IO ()              -- note: functions make great deps
  , dbPool  :: Pool Connection
  , metrics :: IORef Stats                -- mutable state lives IN the env (see below)
  }

newtype App a = App (ReaderT Env IO a)
  deriving newtype (Functor, Applicative, Monad, MonadIO, MonadReader Env)
  -- the newtype'd stack: App-specific instances possible, internals swappable
  -- (deriving newtype = borrow the wrapped type's instances — Part 2's deriving,
  --  with the mechanics in ../typeclasses-newtypes-deriving-via.md)

runApp :: Env -> App a -> IO a
runApp env (App action) = runReaderT action env
```

This is the **ReaderT pattern**, and the Go translation is nearly 1:1 —
which is the fastest way to see that nothing deep is happening:

| Haskell | Go |
| ------- | -- |
| `data Env = Env {...}` | `type Server struct { cfg Config; db *sql.DB; log *slog.Logger }` |
| `doThing :: App Result` | `func (s *Server) DoThing() (Result, error)` |
| `ask` / `asks dbPool` | `s` / `s.db` |
| `runApp env` in `main` | constructing the one `Server{...}` in `main` |

Why `ReaderT Env IO` and not a taller stack? Two conventions worth adopting
wholesale:

- **No `StateT`/`ExceptT` over `IO` in app code.** Mutable state goes in the
  `Env` as an `IORef`/`TVar` (so it survives exceptions and is shared across
  threads, which `StateT`'s thread-it-through state is not); failures use
  `IO` exceptions at the rim and `Either` in the pure core, exactly as
  Part 5 prescribed. The transformer *machinery* from §3 is still what
  you're using — just one layer of it.
- **The `Env` holds functions, not just data** (`logger :: Text -> IO ()`).
  Swapping a dependency for tests means building a different `Env` — no
  mocking framework, just a record with a different function in it.

---

## 6. Where to go next

You can now read most application Haskell. The natural next steps, roughly
in order of payoff:

- **Concurrency** — green threads + STM (`async` library, `TVar`):
  compare-and-swap-free shared state; arguably Haskell's best runtime story
  (and very Go-flavored: `forkIO` ≈ `go`, channels exist too).
- **Property-based testing** — QuickCheck/hedgehog: state invariants
  (`reverse (reverse xs) == xs`), the library generates the cases.
- **The ecosystem staples** — `aeson` (JSON), `servant` or `wai`/`warp`
  (HTTP), `postgresql-simple`/`persistent` (DB) — all heavy users of the
  typeclass machinery in [the deriving-via notes](../typeclasses-newtypes-deriving-via.md).
- **Stronger types** — when ready for the type-level ladder:
  [phantom types](../phantom-types.md) §6 points onward to GADTs and
  `DataKinds`.
- **The classic reading** — the Typeclassopedia (the F/A/M ladder, properly),
  and "The ReaderT design pattern" (Snoyman) for §5 straight from the source.

---

## One-sentence mental model

`Reader`/`State`/`Except` turn the three plumbing rituals — pass the env,
thread the accumulator, bail on error — into monads; transformers stack one
on a base monad with `lift`/`liftIO` bridging the layers; and real apps
settle on the simplest stack that works, `newtype App a = App (ReaderT Env IO a)`:
one env record holding every dependency (Go's service struct, with `ask`
instead of `s.`), supplied once in `main`.

---

[← Structuring programs](06-structuring-programs.md) · [Back to overview](00-overview.md)
