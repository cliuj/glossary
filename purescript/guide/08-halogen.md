# Part 8 — Halogen: the all-PureScript UI framework

[← FFI](07-ffi-typescript.md) · [Next: react-basic-hooks →](09-react-basic-hooks.md)

The first of two framework parts. Halogen is the canonical pure-PureScript
choice: no React underneath, components all the way down, and the type
system threaded through everything — state, actions, even which HTML
attributes an element accepts. This part builds a small app — a counter
plus an async "fetch a quote" button — and [Part 9](09-react-basic-hooks.md)
rebuilds the *same app* on React so you can compare with your own eyes.

If you've written Redux or `useReducer` code, you already know Halogen's
architecture: **state + actions + a reducer**, except the "reducer" can also
perform effects, and every wire is typed.

---

## 1. The anatomy of a component

A Halogen component is four declarations and an assembly call. The whole
shape first, the pieces after:

```purescript
module Main where

import Prelude

import Data.Maybe (Maybe(..))
import Data.Time.Duration (Milliseconds(..))
import Effect (Effect)
import Effect.Aff (delay)
import Effect.Aff.Class (class MonadAff)
import Halogen as H
import Halogen.Aff as HA
import Halogen.HTML as HH
import Halogen.HTML.Events as HE
import Halogen.HTML.Properties as HP
import Halogen.VDom.Driver (runUI)

-- 1. What the component remembers
type State = { count :: Int, quote :: Maybe String }

-- 2. Everything that can happen — a sum type, so handling is EXHAUSTIVE
data Action = Increment | Decrement | LoadQuote

-- 3. State -> HTML, pure (Part 2's RequestState idea will feel natural here)
render :: forall m. State -> H.ComponentHTML Action () m
render state =
  HH.div_
    [ HH.button [ HE.onClick \_ -> Decrement ] [ HH.text "-" ]
    , HH.text (" " <> show state.count <> " ")
    , HH.button [ HE.onClick \_ -> Increment ] [ HH.text "+" ]
    , HH.div_
        [ HH.button [ HE.onClick \_ -> LoadQuote ] [ HH.text "load quote" ]
        , HH.p [ HP.class_ (HH.ClassName "quote") ]
            [ HH.text (case state.quote of
                Nothing -> "no quote yet"
                Just q  -> q)
            ]
        ]
    ]

-- 4. What each action does: update state and/or run effects
handleAction :: forall o m. MonadAff m => Action -> H.HalogenM State Action () o m Unit
handleAction = case _ of
  Increment ->
    H.modify_ \s -> s { count = s.count + 1 }
  Decrement ->
    H.modify_ \s -> s { count = s.count - 1 }
  LoadQuote -> do
    H.liftAff (delay (Milliseconds 300.0))        -- stand-in for a fetch (Part 7 §5)
    H.modify_ _ { quote = Just "simplicity is complicated" }

-- assembly
component :: forall q i o m. MonadAff m => H.Component q i o m
component =
  H.mkComponent
    { initialState: \_ -> { count: 0, quote: Nothing }
    , render
    , eval: H.mkEval H.defaultEval { handleAction = handleAction }
    }

main :: Effect Unit
main = HA.runHalogenAff do
  body <- HA.awaitBody
  runUI component unit body
```

That's a complete, compiling app. Now the pieces, with their React/TS
counterparts.

---

## 2. `render`: your JSX, as functions

`render` is a **pure function from state to HTML** — React's core idea,
which Halogen shares completely. No JSX though: elements are functions from
the `HH` (HTML), `HE` (events), and `HP` (properties) modules:

```purescript
HH.button [ HE.onClick \_ -> Increment ] [ HH.text "+" ]
--         ^ array of props/handlers     ^ array of children
```

```tsx
// TypeScript/React
<button onClick={() => dispatch(Increment)}>+</button>
```

The translation is mechanical: element name → `HH.name`, attributes → first
array, children → second array (`HH.div_` with underscore = "no props"
shorthand). Two things JSX can't offer:

- **It's all just expressions** — `map`, `case`, and every Part 1–4 tool
  work directly; there's no JSX-vs-JS seam, no `{}` switching.
- **Element types are checked** — `HP.href` on a `button` is a compile
  error, not a silent no-op; handlers must produce an `Action`, so a typo'd
  handler can't dispatch nonsense.

Notice `state.quote :: Maybe String` rendering through a `case`: the
"loading/empty/loaded" UI states that TS tracks with booleans-and-hope are
sum types here, and render *must* handle every variant —
[Part 2 §3](02-types-and-data.md#3-sum-types-the-discriminated-union-perfected)'s
`RequestState` pattern is Halogen's bread and butter.

---

## 3. `Action` + `handleAction`: a reducer that can act

All interactivity flows through one sum type. `handleAction` is your
reducer — but where Redux exiles effects to middleware, Halogen's runs in
`HalogenM`, a monad with both state operations *and* effects:

- `H.modify_ :: (State -> State) -> ...` — the state update
  (`s { count = s.count + 1 }` is Part 2's record-update syntax).
- `H.liftAff` / `H.liftEffect` — run any `Aff`/`Effect` from Parts 5–7
  inline: fetch, delay, localStorage, your FFI wrappers. `LoadQuote` above
  *is* the async-thunk pattern, minus the middleware.
- `H.get`, `H.gets _.count` — read state mid-handler when deciding what
  to do.

The payoff over `useReducer`/Redux is the compiler holding both ends:
adding a `Reset` action breaks the build until `handleAction` handles it
(exhaustiveness), and dispatching is just constructing a value, so
`HE.onClick \_ -> Rest` is a compile error, not a silently-ignored string.

The `forall m. MonadAff m =>` in the signatures is
[constraint thinking](02-types-and-data.md#6-type-classes-interfaces-with-explicit-derivable-instances)
applied to architecture: `handleAction` declares "I need async capability"
without naming a concrete monad — the Halogen equivalent of dependency
injection, and the door to swapping in test interpreters later.

---

## 4. `main`: mounting

```purescript
main = HA.runHalogenAff do
  body <- HA.awaitBody          -- wait for DOMContentLoaded, get <body>
  runUI component unit body     -- mount; `unit` is the input (≈ props)
```

`runUI` is `createRoot(document.body).render(<App/>)`. The `unit` is the
component's *input* — Halogen's props, fed to `initialState` (our
`\_ -> ...` ignored it). Bundle with `spago bundle`
([Part 6 §4](06-tooling-and-projects.md#4-the-dev-loop)), load from an
`index.html`, done.

---

## 5. What scaling up looks like

This guide stops at one component, but the path onward is signposted so
real codebases read familiarly:

- **Child components** plug into that `()` slot type you saw in
  `ComponentHTML Action () m` — it's the typed registry of which children
  exist and what they can tell you. Parents receive child *output* through
  it; the wiring is verbose but completely checked.
- **Queries and subscriptions** (the `q` in `H.Component q i o m`) cover
  parent-commands-child and external-event-streams (timers, websockets,
  window events).
- **Lifecycles** are two more fields on `defaultEval` (`initialize`,
  `finalize`) dispatching ordinary `Action`s — no `useEffect` dependency
  arrays, just "when mounted, do `Initialize`."
- Many teams keep *page-level* components few and fat, with plain
  `State -> HTML` functions (not components!) doing the decomposition —
  cheap, stateless, and exactly the "extract a render helper" reflex from
  React, minus the hooks-rules anxiety.

---

## One-sentence mental model

Halogen is the Redux architecture with the compiler holding every wire:
state is a record, *everything that can happen* is a sum type, render is a
pure state→HTML function in plain PureScript (no JSX seam), and the reducer
runs in a monad where `modify_` updates and `liftAff` performs — so
forgotten cases, misspelled actions, and impossible UI states are compile
errors instead of bug reports.

---

[← FFI](07-ffi-typescript.md) · [Next: react-basic-hooks →](09-react-basic-hooks.md)
