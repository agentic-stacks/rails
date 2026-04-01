# Asset Pipeline Decision Guide

Choose between Propshaft + importmap, Propshaft + esbuild, and Sprockets for your Rails app.

---

## Comparison Table

| Criterion | Propshaft + importmap | Propshaft + esbuild | Sprockets 4 |
|---|---|---|---|
| **Rails 8 default** | Yes | No | No |
| **Node required** | No | Yes | No |
| **Build step** | None | Yes (`bin/dev`) | None (dev) / Yes (prod) |
| **Tree shaking** | No | Yes | No |
| **TypeScript** | No (transpile manually) | Yes (native) | No |
| **JSX / React / Vue** | No | Yes | No |
| **CSS bundling** | Basic (Propshaft passthrough) | Via PostCSS / Tailwind CLI | Full (Sass, Less) |
| **Fingerprinting** | Yes | Yes | Yes |
| **Source maps** | Limited | Yes | Limited |
| **Complexity** | Low | Medium | Medium–High |
| **NPM ecosystem access** | Partial (CDN pins) | Full | Partial |
| **Tailwind CSS** | Via standalone CLI | Via PostCSS or CLI | Via `tailwindcss-rails` |
| **Sprockets compatibility** | None | None | N/A |

---

## Option Details

### Propshaft + importmap (Rails 8 default)

Propshaft is a minimal asset pipeline: it fingerprints and serves files without compilation. Importmap-rails manages JavaScript modules via browser-native ES module imports, pinning packages either to a CDN (jsDelivr, unpkg) or to `vendor/javascript/`.

**Strengths:**
- No Node, no build step, no `node_modules`
- Fast CI — no `npm install` or JS compilation
- Simple mental model: files go in, fingerprinted files come out
- Works well with Stimulus and Hotwire (the Rails 8 defaults)

**Weaknesses:**
- No bundling — every import is a separate HTTP request (mitigated by HTTP/2)
- No TypeScript or JSX without an external transpiler
- NPM packages must be vendored or pinned to a CDN — complex packages with many sub-dependencies are awkward
- Tree shaking is not available; unused code ships to the browser

**When to avoid:**
- Apps with complex front-end requirements (React SPA, Vue components, TypeScript codebase)
- Teams that rely heavily on NPM toolchain (lint, format, test in JS)

---

### Propshaft + esbuild (via jsbundling-rails)

jsbundling-rails provides a thin wrapper around esbuild (or rollup/webpack). JavaScript is compiled by esbuild and the output is placed in `app/assets/builds/`, where Propshaft fingerprints and serves it. Requires Node and runs via a `bin/dev` process.

**Strengths:**
- Full NPM ecosystem access — install any package with `npm install`
- TypeScript, JSX, and modern ES features via esbuild
- Tree shaking reduces bundle size
- Source maps for debugging
- Fast builds (esbuild is written in Go)

**Weaknesses:**
- Requires Node in development and CI
- Two processes in development (`rails server` + `esbuild --watch`)
- More moving parts: `package.json`, `node_modules`, `Procfile.dev`, build output directory
- CSS bundling is separate (cssbundling-rails or Tailwind standalone CLI)

**When to avoid:**
- Apps that want zero Node dependency
- Simple CRUD apps where the complexity cost is not justified

---

### Sprockets 4

The classic Rails asset pipeline. Handles JavaScript, CSS, and images through a compilation and concatenation pipeline with a manifest.

**Strengths:**
- Well-understood and documented
- Large ecosystem of gems that hook into it (e.g., `sass-rails`, `uglifier`)
- Handles CSS preprocessing (Sass) natively
- Familiarity for teams with Rails 5/6 backgrounds

**Weaknesses:**
- Not the Rails default since 7.x; Propshaft is the direction
- Slow asset compilation on large apps
- No tree shaking, no ES modules natively
- `sprockets-rails` is in maintenance mode
- Complex debugging when the manifest or cache gets into a bad state
- Incompatible with Propshaft — you choose one or the other

**When to avoid:**
- New apps on Rails 7.2+ — use Propshaft instead
- Apps that need modern JS tooling — use esbuild with Propshaft

---

## Recommendations by Use Case

| Use Case | Recommended Setup | Rationale |
|---|---|---|
| New Rails 8 app, server-rendered HTML, Stimulus/Hotwire | Propshaft + importmap | Zero Node, matches Rails defaults, lowest complexity |
| New Rails 8 app with TypeScript or JSX | Propshaft + esbuild | Full NPM access, tree shaking, source maps |
| New Rails 8 app with React or Vue SPA | Propshaft + esbuild | Required for JSX and component compilation |
| Existing Rails 7 app using Sprockets, low JS complexity | Keep Sprockets (short term), plan Propshaft migration | Migration cost rarely justified without a feature driver |
| Existing Rails 7 app using Sprockets, high JS complexity | Migrate to Propshaft + esbuild | esbuild gives modern tooling; migration is non-trivial but worth it |
| CI/CD with strict Node version control | Propshaft + importmap | Eliminates Node as a CI dependency |

---

## Migration Path: Sprockets to Propshaft

Migrating is a multi-step process. Plan for 1–3 days of work depending on app complexity.

1. **Audit asset usage:**
   ```bash
   grep -r "asset_path\|asset_url\|image_tag\|stylesheet_link_tag\|javascript_include_tag" app/views app/helpers
   ```

2. **Add Propshaft, remove Sprockets:**
   ```ruby
   # Gemfile
   gem "propshaft"
   # Remove: gem "sprockets-rails", gem "sprockets", gem "sass-rails"
   ```

3. **Remove Sprockets configuration:**
   - Delete `config/initializers/assets.rb` or move relevant config to `config/application.rb`
   - Remove `config.assets.paths`, `config.assets.precompile` — Propshaft does not use these

4. **Move or rename assets:**
   - Propshaft serves files directly without compilation — Sass files must be pre-compiled or replaced with plain CSS
   - Move any `app/assets/stylesheets/*.scss` to plain `.css` or add a CSS bundler

5. **Update asset helpers:**
   - `asset_path` and `asset_url` work with Propshaft but only for files in the Propshaft load path
   - Remove `:all` from `javascript_include_tag` calls

6. **Precompile and verify:**
   ```bash
   bin/rails assets:clobber
   bin/rails assets:precompile
   bin/rails assets:reveal  # Propshaft command: lists all served assets
   ```

7. **Test in production mode locally:**
   ```bash
   RAILS_ENV=production bin/rails server
   ```
