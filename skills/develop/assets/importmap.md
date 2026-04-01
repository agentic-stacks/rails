# Import Maps: JavaScript Without a Bundler

Import Maps let browsers load JavaScript ES modules by name, mapping bare specifiers like `"@hotwired/turbo"` to URLs. Rails ships `importmap-rails` as the default JavaScript strategy for new apps. No Node.js, no Yarn, no webpack.

## How It Works

The browser's native `<script type="importmap">` tag defines a JSON mapping of module names to URLs. `importmap-rails` generates this tag and manages pinning packages to CDN or local files.

```html
<!-- Rendered by <%= javascript_importmap_tags %> -->
<script type="importmap">
{
  "imports": {
    "application": "/assets/application-abc123.js",
    "@hotwired/turbo-rails": "/assets/turbo.es2017-esm-def456.js",
    "@hotwired/stimulus": "/assets/stimulus.esm-789abc.js"
  }
}
</script>
<script type="module">import "application"</script>
```

---

## Setup

### New App (Automatic)

Rails 8 and Rails 7 both default to Import Maps:

```bash
rails new myapp              # Import Maps included
```

### Add to Existing App

```bash
bundle add importmap-rails
bin/rails importmap:install
```

`importmap:install` adds:
- `config/importmap.rb` — pin definitions
- `app/javascript/application.js` — entry point
- `<%= javascript_importmap_tags %>` to the layout

---

## config/importmap.rb

This file defines all pins. Run any `bin/importmap` command to update it.

```ruby
# config/importmap.rb

# Pin the app entry point (served from app/javascript/)
pin "application"

# Pin a package from a CDN
pin "@hotwired/turbo-rails", to: "turbo.es2017-esm.js"
pin "@hotwired/stimulus", to: "stimulus.esm.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"

# Pin a local file in app/javascript/
pin "local_lib", to: "local_lib.js"

# Pin all controllers (glob)
pin_all_from "app/javascript/controllers", under: "controllers"
```

---

## Pinning Packages

### From jsDelivr CDN (default)

```bash
bin/importmap pin lodash
# Adds to config/importmap.rb:
# pin "lodash", to: "https://cdn.jsdelivr.net/npm/lodash@4.17.21/lodash.js"
```

### From Unpkg

```bash
bin/importmap pin lodash --from unpkg
```

### Download Locally

Pin and download to `vendor/javascript/` so the app works without CDN access:

```bash
bin/importmap pin lodash --download
# Saves to vendor/javascript/lodash.js
# Pins as: pin "lodash", to: "lodash.js"
```

### Pin a Specific Version

```bash
bin/importmap pin alpinejs@3.13.3
```

### Unpin

```bash
bin/importmap unpin lodash
```

### Audit for Outdated Pins

```bash
bin/importmap outdated
```

---

## app/javascript/application.js

The entry point. Import everything your app needs:

```javascript
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "controllers"
// import "channels"   // uncomment for Action Cable
```

---

## Stimulus Auto-Loading

`pin_all_from` pins every file in a directory. Combined with `stimulus-loading`, controllers are registered automatically based on filename:

```ruby
# config/importmap.rb
pin_all_from "app/javascript/controllers", under: "controllers"
```

```javascript
// app/javascript/controllers/index.js
import { application } from "./application"
import { eagerLoadControllersFrom } from "@hotwired/stimulus-loading"
eagerLoadControllersFrom("controllers", application)
```

File `app/javascript/controllers/hello_controller.js` is automatically registered as identifier `hello`.

---

## Using Imported Modules

```javascript
// app/javascript/application.js
import { debounce } from "lodash"
import confetti from "canvas-confetti"

window.addEventListener("turbo:load", () => {
  // ...
})
```

---

## Limitations

Know these before choosing Import Maps:

| Limitation | Details |
|---|---|
| No tree shaking | Entire packages are loaded, even unused code. Avoid large utility libraries. |
| No TypeScript | Browsers run ES modules natively — no transpilation step. Use `.js` only. |
| No JSX / React | React requires a build step for JSX. Use jsbundling for React. |
| CDN dependency | Default pins point to jsDelivr. Use `--download` for fully offline builds. |
| Browser support | Import Maps are supported in all modern browsers. IE is not supported. |
| No `require()` (CJS) | CommonJS modules (`require()`/`module.exports`) are not supported. Must use ESM packages. |
| Code splitting complexity | Manual — no automatic chunking. |

---

## Preloading Modules

By default, imported modules are loaded on demand. Preload critical modules to avoid waterfall fetches:

```ruby
# config/importmap.rb
pin "@hotwired/turbo-rails", to: "turbo.es2017-esm.js", preload: true
pin "@hotwired/stimulus",    to: "stimulus.esm.js",     preload: true
```

This adds `<link rel="modulepreload">` tags to `<head>`, telling the browser to fetch these modules early.

---

## Integrity / Subresource Integrity (SRI)

When pinning from CDN in production, add `integrity` hashes to protect against CDN tampering:

```bash
bin/importmap pin alpinejs --download   # local copy is safer
```

For CDN pins, `bin/importmap pin` fetches integrity hashes automatically and stores them in `config/importmap.rb`:

```ruby
pin "alpinejs", to: "https://cdn.jsdelivr.net/npm/alpinejs@3.13.3/dist/module.esm.js",
    integrity: "sha256-..."
```

---

## Development vs Production

In development, `javascript_importmap_tags` serves files directly from `app/javascript/` and `vendor/javascript/` — no precompile needed.

In production, Propshaft fingerprints the files and the import map URLs point to fingerprinted paths. Run precompile as part of your deploy:

```bash
RAILS_ENV=production bin/rails assets:precompile
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `Uncaught TypeError: Failed to resolve module specifier` | Package not pinned | Run `bin/importmap pin <package>` |
| Old version still loading | Browser cache | Hard reload; check `bin/importmap outdated` |
| Controller not registered | File not in pinned directory | Check `pin_all_from` path in `config/importmap.rb` |
| CJS package fails | Package uses `require()` | Use jsbundling instead; find an ESM-compatible version |
| 404 for pinned CDN file | CDN unavailable or version removed | Use `--download` to vendor locally |

---

## Related Skills

- `propshaft.md` — asset serving (pairs with Import Maps)
- `jsbundling.md` — when you need TypeScript, React, or tree shaking
- `../views/stimulus.md` — Stimulus controllers loaded via Import Maps
