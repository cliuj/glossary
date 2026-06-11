# Part 6 — Tooling and projects

[← Effect and Aff](05-effect-and-aff.md) · [Next: FFI — using JS/TS libraries →](07-ffi-typescript.md)

The language tour is done; this part is the working environment. Good news:
PureScript lives *inside* the toolchain you already run — the compiler and
package manager install from npm, the bundler is esbuild, and the output is
ES modules in a folder. Mapping table first, details after.

*(Everything here is spago 1.x — the `spago.yaml` era. Older tutorials
showing `spago.dhall` or `packages.dhall` files are the previous generation;
the concepts match, the files don't.)*

---

## 1. The toolchain, mapped

| Job | TypeScript | PureScript | Notes |
| --- | ---------- | ---------- | ----- |
| compiler | `tsc` | **`purs`** | emits readable ES modules into `output/` |
| package manager + build runner | `npm` + `tsc -b` scripts | **`spago`** | one tool: deps, build, run, test, repl, bundle |
| manifest | `package.json` + `tsconfig.json` | **`spago.yaml`** | one file does both jobs |
| bundler | esbuild/vite/webpack | **esbuild** | literally the same tool — spago shells out to it |
| editor brain | tsserver | **`purs ide`** | via the *PureScript IDE* VS Code extension; rebuild-on-save, types on hover, auto-imports |
| REPL | `node` / ts-node | **`spago repl`** | first-class: try any expression, `:type` anything |

Everything installs from npm, so a PureScript project can be bootstrapped
with the reflexes you already have:

```bash
npm install --save-dev purescript spago esbuild
npx spago init
npx spago run            # builds and runs main
```

---

## 2. `spago.yaml`: the manifest

`spago init` generates this (trimmed):

```yaml
package:
  name: myapp
  dependencies:          # ← package.json "dependencies"
    - console
    - effect
    - prelude
  test:
    main: Test.Main
    dependencies: []
workspace:
  packageSet:
    registry: 77.5.0     # ← the lockfile-ish part — see below
  extraPackages: {}
```

`spago install aff arrays` adds dependencies (editing the yaml for you,
like `npm install --save`). The commands you'll live in:

```bash
spago build              # compile everything to output/
spago run                # build + run the main module
spago test               # build + run Test.Main
spago repl               # the REPL, with your project loaded
spago bundle --bundle-type app --outfile app.js   # esbuild, for the browser
```

### Package sets: the part npm doesn't have

That `registry: 77.5.0` line is a **package set** — a curated snapshot of
the PureScript registry in which **every package version is known to
compile together**. You don't pick versions per-package and you don't solve
constraints; you pick one set, and `dependencies` is just *names*. Upgrading
the ecosystem = bumping one number.

```typescript
// The npm experience this replaces:
// "why did my transitive dep's minor bump break the build at 2am"
```

The tradeoff is real but mild: you get the set's versions (pinning/newer
versions go in `extraPackages`). For application work it's strictly less
yak-shaving. Browsing what's *in* the ecosystem happens on **Pursuit**
(pursuit.purescript.org) — docs plus the killer feature: **search by type
signature**. Want "split an array in two by a predicate"? Search
`(a -> Boolean) -> Array a -> { yes :: Array a, no :: Array a }` and
`partition` comes back. It's how PureScript developers actually find
functions; train the reflex early.

---

## 3. Project anatomy

```
myapp/
├── spago.yaml
├── package.json         # npm deps live alongside — Part 7 uses this
├── src/
│   └── Main.purs        # module Main — `main :: Effect Unit` lives here
├── test/
│   └── Test/Main.purs
└── output/              # compiled ES modules (gitignore it)
    └── Main/index.js
```

One file = one module ([naming](01-reading-purescript.md#8-names-and-comments):
`src/Data/Validation.purs` holds `module Data.Validation`), and a module's
header controls its public API — there's no per-declaration `export`
keyword:

```purescript
module Order
  ( Order            -- export the type but NOT its constructor...
  , mkOrder          -- ...so this validating function is the only way in
  , total
  ) where
```

That "type without constructor" trick is the **smart constructor** pattern —
invalid `Order`s become unrepresentable outside the module, because only
`mkOrder :: Items -> Either OrderError Order` can make one. It's the
module-system counterpart to Part 2's "illegal states don't compile."
Omit the export list and everything is public (fine while learning).

### The output is genuinely readable

`output/` is not minified artifact soup — it's the JS you'd write, and
reading it is a legitimate learning technique. Part 1's `add`, compiled:

```javascript
// output/Check01/index.js — actual emitted code
var add = function (x) {
    return function (y) {
        return x + y | 0;        // | 0 is the Int-ness, enforced
    };
};
var add3 = /* #__PURE__ */ add(3);
```

Currying, partial application, `Int` semantics — all visible, all debuggable
with normal browser devtools (source maps available via
`spago bundle --source-maps`). When you wonder "what does this compile to,"
the answer is one `cat` away — and Part 7's FFI is built on exactly this
transparency.

---

## 4. The dev loop

- **Editor-driven:** install the **PureScript IDE** extension (VS Code) —
  it talks to `purs ide` for on-save rebuilds, inline errors, hover types,
  go-to-definition, and auto-imports. This replaces a `--watch` flag for
  the compile-feedback loop (spago 1.x deliberately has none; an external
  watcher like `watchexec -e purs -- spago build` covers the terminal case).
- **REPL-driven:** `spago repl` loads your project; `import MyModule` and
  poke at real functions. `:type expr` answers most "what is this" questions
  faster than hover. (`:paste` for multi-line input, Ctrl-D to finish.)
- **Browser-driven:** `spago bundle` (esbuild underneath, `--minify` for
  release) produces a script your `index.html` loads — or any vite/esbuild
  setup can import `output/Main/index.js` directly as an ES module and own
  the serving/HMR story itself. Parts 8–9 use this.

A note on compile speed: clean builds compile *the world* (your package
set's deps) and feel slow; incremental builds are fast and the editor loop
is instant-feeling. Don't judge the dev experience by the first
`spago build`.

---

## One-sentence mental model

`purs` is your `tsc`, `spago` is your npm-plus-build-scripts driving one
`spago.yaml`, esbuild is *literally* your esbuild, versions come as one
curated set instead of a constraint solve, Pursuit searches the ecosystem by
type signature — and the compiled output is readable ES modules you're
encouraged to go look at.

---

[← Effect and Aff](05-effect-and-aff.md) · [Next: FFI — using JS/TS libraries →](07-ffi-typescript.md)
