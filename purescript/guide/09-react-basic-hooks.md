# Part 9 — react-basic-hooks: PureScript on real React

[← Halogen](08-halogen.md) · [Back to overview](00-overview.md)

The second framework option — and for a React developer, the uncanny one:
**it's actual React** (the npm package, in your bundle, with its scheduler,
devtools, and ecosystem), driven from PureScript through typed bindings.
Same app as [Part 8](08-halogen.md): counter plus async quote. Then the
comparison you came for.

```bash
npm install react react-dom
spago install react-basic react-basic-hooks react-basic-dom
```

---

## 1. The same app, hooks edition

```purescript
module Main where

import Prelude

import Data.Maybe (Maybe(..), fromMaybe)
import Data.Time.Duration (Milliseconds(..))
import Data.Tuple.Nested ((/\))
import Effect (Effect)
import Effect.Aff (delay, launchAff_)
import Effect.Class (liftEffect)
import Effect.Exception (throw)
import React.Basic.DOM as R
import React.Basic.DOM.Client (createRoot, renderRoot)
import React.Basic.Events (handler_)
import React.Basic.Hooks (Component, component, useState)
import React.Basic.Hooks as React
import Web.DOM.NonElementParentNode (getElementById)
import Web.HTML (window)
import Web.HTML.HTMLDocument (toNonElementParentNode)
import Web.HTML.Window (document)

mkApp :: Component Unit
mkApp = component "App" \_ -> React.do
  count /\ setCount <- useState 0
  quote /\ setQuote <- useState Nothing
  let
    loadQuote :: Effect Unit
    loadQuote = launchAff_ do
      delay (Milliseconds 300.0)                  -- stand-in for a fetch
      liftEffect (setQuote \_ -> Just "simplicity is complicated")
  pure
    ( R.div_
        [ R.button { onClick: handler_ (setCount (_ - 1)), children: [ R.text "-" ] }
        , R.text (" " <> show count <> " ")
        , R.button { onClick: handler_ (setCount (_ + 1)), children: [ R.text "+" ] }
        , R.div_
            [ R.button { onClick: handler_ loadQuote, children: [ R.text "load quote" ] }
            , R.p { className: "quote"
                  , children: [ R.text (fromMaybe "no quote yet" quote) ]
                  }
            ]
        ]
    )

main :: Effect Unit
main = do
  doc <- document =<< window
  maybeEl <- getElementById "app" (toNonElementParentNode doc)
  case maybeEl of
    Nothing -> throw "no #app element"
    Just el -> do
      app <- mkApp
      root <- createRoot el
      renderRoot root (app unit)
```

If you squint past syntax, this *is* the React function component you'd
write in TS:

```tsx
// TypeScript — the same component
function App() {
  const [count, setCount] = useState(0);
  const [quote, setQuote] = useState<string | null>(null);
  const loadQuote = async () => {
    await sleep(300);
    setQuote("simplicity is complicated");
  };
  return (
    <div>
      <button onClick={() => setCount((c) => c - 1)}>-</button>
      {` ${count} `}
      <button onClick={() => setCount((c) => c + 1)}>+</button>
      <div>
        <button onClick={loadQuote}>load quote</button>
        <p className="quote">{quote ?? "no quote yet"}</p>
      </div>
    </div>
  );
}
```

---

## 2. The translation dictionary

| React/TS | react-basic-hooks | Notes |
| -------- | ----------------- | ----- |
| `function App() {...}` | `component "App" \props -> React.do ...` | creating a component is an `Effect` (it registers with React) — hence `mkApp` and the `app <- mkApp` in `main` |
| hook body | `React.do` | a *qualified do*: do-notation for the hooks context — and the hooks **rules** (fixed order, no hooks in `if`) are enforced by its types, not a lint plugin |
| `const [x, setX] = useState(0)` | `x /\ setX <- useState 0` | `/\` is a tuple pattern — the same destructuring; `setX` takes an *updater function* (`setX (_ + 1)`), the `setX(x => x+1)` form |
| JSX | `R.div_`, `R.button {...}` | props are a record — `children`, `className`, `onClick` keep their React names |
| `onClick={fn}` | `onClick: handler_ fn` | `handler_` wraps an `Effect Unit`; `handler` (no underscore) gives you the event object, typed |
| `async` handler | `launchAff_ do ...` | [Part 5](05-effect-and-aff.md)'s launcher, verbatim — state setters inside need `liftEffect` |
| `useEffect(fn, deps)` | `useEffect deps (...)` | deps are typed and equality-checked — no "exhaustive-deps" guesswork |
| `createRoot(el).render(<App/>)` | `createRoot el >>= \r -> renderRoot r (app unit)` | React 18 client API, mirrored |

Things you keep by choosing React: the entire component ecosystem (every
React library crosses one [Part 7](07-ffi-typescript.md) FFI wrapper away —
`ReactComponent props` is the bridge type for both importing theirs and
`reactComponent`-exporting yours), React devtools, concurrent rendering,
and your own muscle memory. Things PureScript adds on top: `quote` is a
real `Maybe` (not `string | null` hoping strictNullChecks holds), every
handler is typed all the way down, and Parts 1–5's language — sum types in
state, `Either` pipelines in logic, `Aff` instead of promise juggling —
runs *inside* the components.

---

## 3. Halogen or React? The actual decision

Both are production-quality; both had every example in these two parts
compiled against current package sets. The honest discriminators:

| | **Halogen** | **react-basic-hooks** |
| --- | --- | --- |
| Underneath | its own VDOM, all PureScript | real React from npm |
| Architecture | one: state/Action/handleAction (Elm-ish, Redux-ish) | React's: hooks, effects, context |
| Ecosystem | PureScript-only (smaller, but designed-for) | all of React, one FFI wrapper away |
| Type-safety ceiling | highest — components, children, queries all typed | high inside components; FFI honesty at ecosystem edges |
| Uses your React experience | conceptually (render = f(state)) | directly — hooks, props, devtools transfer |
| Team onboarding | learn Halogen | TS/React colleagues can *read* it day one |
| Bundle | ~your code + Halogen | + React runtime |
| Docs/community | Halogen guide + Real World Halogen | thinner docs; leans on React knowledge + Pursuit |

A defensible rule of thumb for your situation (TS/React background,
solo-ish frontend projects):

- Pick **react-basic-hooks** when the project will lean on existing React
  libraries (component kits, charts, maps), when TS colleagues may touch
  it, or when you want PureScript's language with minimum framework
  novelty. It's also the gentler first PureScript project — one new thing
  at a time.
- Pick **Halogen** when the app is self-contained (forms, dashboards,
  editors you'd build from scratch anyway), when you want the strongest
  compiler grip end-to-end, or when the point is to learn how UI
  architecture feels with *no* escape hatches. Going all-in teaches more
  PureScript per week.

And it's genuinely fine to start with one small page in each — Parts 8 and
9 are intentionally the same app so the diff is small enough to feel.

---

## Where to go next

- **HTTP for real:** `affjax-web` (an `Aff` HTTP client) or a thin
  `fetch` FFI wrapper ([Part 7 §5](07-ffi-typescript.md#5-promises--aff-the-async-bridge))
  — plus `argonaut`/`codec-argonaut` to decode responses, zod-style.
- **Routing:** `routing-duplex` — typed route ↔ URL codecs (the
  bidirectionality means dead links don't compile).
- **Reading material:** *PureScript by Example* (the community-maintained
  book), the Halogen guide, Thomas Honeyman's *Real World Halogen*, and
  Jordan's PureScript Reference for the deep cuts.
- **The Haskell guide next door** ([`../../haskell/guide/`](../../haskell/guide/00-overview.md)):
  Parts 6–7 there (type classes in depth, monad transformers, ReaderT) are
  the natural "intermediate tier" for PureScript too — the concepts carry
  over almost verbatim, and `Reader`-shaped architecture shows up in larger
  Halogen apps.

---

## One-sentence mental model

react-basic-hooks is your React knowledge with PureScript's type system
riding along — `useState` destructures with `/\`, `React.do` makes the hook
rules type-checked instead of linted, handlers are `Effect`s and async is
`Aff` — so the choice against Halogen is really "React's ecosystem and
familiarity" versus "one fully-typed architecture with no escape hatches,"
and the two parts you just read are the same app so you can diff the feel.

---

[← Halogen](08-halogen.md) · [Back to overview](00-overview.md)
