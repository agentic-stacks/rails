# jsbundling-rails: JavaScript Bundling with esbuild, Rollup, or Webpack

`jsbundling-rails` integrates Node.js-based JavaScript bundlers into Rails. Use it when Import Maps are insufficient — typically when you need TypeScript, React/Vue/Svelte, tree shaking, or the wider npm ecosystem.

## When to Use jsbundling

| Need | Use |
|---|---|
| TypeScript | jsbundling |
| React, Vue, Svelte | jsbundling |
| Tree shaking (dead code elimination) | jsbundling |
| Large npm packages with CJS modules | jsbundling |
| Code splitting / lazy loading | jsbundling (esbuild or rollup) |
| Vanilla JS, Stimulus, Turbo only | Import Maps (no bundler needed) |

---

## Bundler Comparison

| Bundler | Build Speed | Config Complexity | Best For |
|---|---|---|---|
| **esbuild** | Very fast (Go-based) | Minimal | Most Rails apps |
| **rollup** | Moderate | Moderate | Library authors, tree shaking |
| **webpack** | Slow | High | Complex SPAs, legacy setups |

**Recommendation: use esbuild for new Rails apps that need a bundler.**

---

## Installation

### New App

```bash
rails new myapp --javascript esbuild
rails new myapp --javascript rollup
rails new myapp --javascript webpack
```

This runs the installer automatically.

### Add to Existing App

```bash
bundle add jsbundling-rails
bin/rails javascript:install:esbuild   # or rollup, or webpack
```

The installer:
1. Adds `jsbundling-rails` to Gemfile
2. Creates `package.json` with bundler dependency
3. Adds a `build` script to `package.json`
4. Adds `app/assets/builds/` to the asset load path
5. Creates or updates `Procfile.dev`
6. Adds `app/assets/builds/` to `.gitignore`

---

## How It Works

The bundler reads `app/javascript/application.js`, bundles all imports, and writes the output to `app/assets/builds/application.js`. Propshaft (or Sprockets) then serves that file.

```
app/javascript/application.js
  └─ imports ...
       └─ bundler (esbuild) ──► app/assets/builds/application.js
                                        │
                               Propshaft fingerprints
                                        │
                             public/assets/application-abc.js
```

---

## package.json

The installer generates a `package.json` with a `build` script:

```json
{
  "name": "myapp",
  "private": true,
  "dependencies": {
    "@hotwired/stimulus": "^3.2.2",
    "@hotwired/turbo-rails": "^8.0.12",
    "esbuild": "^0.24.0"
  },
  "scripts": {
    "build": "esbuild app/javascript/*.* --bundle --sourcemap --outdir=app/assets/builds --public-path=/assets"
  }
}
```

### Common esbuild Flags

```json
"build": "esbuild app/javascript/*.* --bundle --sourcemap --format=esm --splitting --outdir=app/assets/builds --public-path=/assets"
```

| Flag | Effect |
|---|---|
| `--bundle` | Bundle all imports into one (or split) file |
| `--sourcemap` | Generate `.js.map` files for debugging |
| `--minify` | Minify in production |
| `--format=esm` | Output ES modules |
| `--splitting` | Enable code splitting (requires `--format=esm`) |
| `--target=es2020` | Set JS target for transpilation |

### Minify in Production

```json
"scripts": {
  "build": "esbuild app/javascript/*.* --bundle --sourcemap --outdir=app/assets/builds",
  "build:prod": "esbuild app/javascript/*.* --bundle --minify --outdir=app/assets/builds"
}
```

Or use an environment variable:

```json
"build": "esbuild app/javascript/*.* --bundle --sourcemap $([ \"$RAILS_ENV\" = \"production\" ] && echo '--minify' || echo '') --outdir=app/assets/builds"
```

---

## Development: bin/dev

In development, run both the Rails server and the JS watcher:

```bash
bin/dev
```

`Procfile.dev`:

```
web: bin/rails server -p 3000
js: yarn build --watch
```

The `--watch` flag rebuilds `app/assets/builds/application.js` whenever a source file in `app/javascript/` changes. Rails picks up the new file immediately (no server restart).

### Install Yarn (if needed)

```bash
npm install -g yarn
yarn install   # install dependencies from package.json
```

Or use npm directly — replace `yarn build` with `npm run build` in `Procfile.dev`.

---

## App Entry Point

```javascript
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "./controllers"

// React example:
import React from "react"
import { createRoot } from "react-dom/client"
import App from "./components/App"

document.addEventListener("turbo:load", () => {
  const container = document.getElementById("react-root")
  if (container) {
    createRoot(container).render(<App />)
  }
})
```

### TypeScript

esbuild handles TypeScript natively (strips types, does not type-check):

```bash
yarn add --dev typescript
# Rename files to .ts / .tsx
```

Update `package.json`:

```json
"build": "esbuild app/javascript/*.* --bundle --sourcemap --outdir=app/assets/builds --loader:.tsx=tsx"
```

Add a `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "react-jsx"
  }
}
```

For type-checking (separate from the build):

```json
"scripts": {
  "build": "esbuild ...",
  "typecheck": "tsc --noEmit"
}
```

---

## Production Build

The Rails assets:precompile task runs `yarn build` automatically before fingerprinting:

```bash
RAILS_ENV=production bin/rails assets:precompile
```

This:
1. Runs `yarn build` (or `npm run build`)
2. Writes bundles to `app/assets/builds/`
3. Propshaft fingerprints everything in `app/assets/` and writes to `public/assets/`

Ensure `node` and `yarn` (or `npm`) are available in your production/CI environment.

---

## Stimulus with jsbundling

Install Stimulus via npm instead of importmap:

```bash
yarn add @hotwired/stimulus
```

```javascript
// app/javascript/controllers/application.js
import { Application } from "@hotwired/stimulus"
const application = Application.start()
export { application }

// app/javascript/controllers/index.js
import { application } from "./application"
import HelloController from "./hello_controller"
application.register("hello", HelloController)
```

Or use the webpack/esbuild glob loader:

```javascript
// app/javascript/controllers/index.js
import { application } from "./application"

// Vite/esbuild glob — requires bundler support
const controllers = import.meta.glob("./**/*_controller.js", { eager: true })
for (const [path, module] of Object.entries(controllers)) {
  const name = path.replace("./", "").replace("_controller.js", "").replace("/", "--")
  application.register(name, module.default)
}
```

---

## Rollup Setup

```json
"scripts": {
  "build": "rollup -c rollup.config.js"
}
```

```javascript
// rollup.config.js
import resolve from "@rollup/plugin-node-resolve"
import commonjs from "@rollup/plugin-commonjs"

export default {
  input: "app/javascript/application.js",
  output: {
    file: "app/assets/builds/application.js",
    format: "esm",
    sourcemap: true
  },
  plugins: [resolve(), commonjs()]
}
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `application.js not found` in production | Build not run before precompile | Ensure `yarn build` succeeds in CI |
| JS changes not reflected in dev | `bin/dev` not running | Start with `bin/dev` instead of `bin/rails server` |
| `Cannot find module` | Package not installed | Run `yarn install` |
| TypeScript errors block build | esbuild strips types, never fails on type errors | Run `yarn typecheck` separately |
| Large bundle size | All imports bundled | Add `--splitting --format=esm` for code splitting |

---

## Related Skills

- `importmap.md` — no-bundler alternative for simpler apps
- `cssbundling.md` — CSS bundling (Tailwind, Sass) — often used alongside jsbundling
- `propshaft.md` — asset serving (pairs with jsbundling output)
- `../views/stimulus.md` — Stimulus controllers
