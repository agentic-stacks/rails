# cssbundling-rails: Tailwind, PostCSS, and Sass

`cssbundling-rails` integrates Node.js CSS processors into Rails. It runs a CSS build tool as a separate process alongside the Rails server, writes output to `app/assets/builds/`, and Propshaft (or Sprockets) serves the result.

## When to Use cssbundling

| Need | Use |
|---|---|
| Tailwind CSS (utility classes) | cssbundling (Tailwind) |
| Sass / SCSS | cssbundling (Sass) |
| PostCSS plugins (autoprefixer, nesting, etc.) | cssbundling (PostCSS) |
| Bootstrap via Sass | cssbundling (Sass) |
| Plain CSS, no preprocessing | Propshaft alone (no cssbundling needed) |
| Tailwind CDN (prototyping only) | CDN `<script>` tag — no build needed |

---

## Installation

### New App

```bash
rails new myapp --css tailwind
rails new myapp --css bootstrap   # bootstrap via sass
rails new myapp --css postcss
rails new myapp --css sass
```

### Add to Existing App

```bash
bundle add cssbundling-rails
bin/rails css:install:tailwind
bin/rails css:install:bootstrap
bin/rails css:install:postcss
bin/rails css:install:sass
```

Each installer:
1. Adds `cssbundling-rails` gem
2. Installs Node packages via `yarn add`
3. Creates a CSS entry point in `app/assets/stylesheets/`
4. Adds a `build:css` script to `package.json`
5. Adds `app/assets/builds/` to the asset load path
6. Updates `Procfile.dev` with a CSS watcher

---

## Tailwind CSS

### Setup

```bash
bin/rails css:install:tailwind
# Installs: tailwindcss, @tailwindcss/forms, @tailwindcss/typography (optional)
```

`package.json` build script added:

```json
"build:css": "tailwindcss -i ./app/assets/stylesheets/application.tailwind.css -o ./app/assets/builds/application.css --minify"
```

### Entry Point

```css
/* app/assets/stylesheets/application.tailwind.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Add custom CSS below */
@layer components {
  .btn {
    @apply px-4 py-2 rounded font-medium transition-colors;
  }
  .btn-primary {
    @apply bg-blue-600 text-white hover:bg-blue-700;
  }
}
```

### tailwind.config.js

```javascript
// tailwind.config.js
module.exports = {
  content: [
    "./app/views/**/*.html.erb",
    "./app/views/**/*.html.haml",
    "./app/javascript/**/*.js",
    "./app/components/**/*.html.erb",
    "./app/helpers/**/*.rb"
  ],
  theme: {
    extend: {
      colors: {
        brand: {
          50:  "#eff6ff",
          500: "#3b82f6",
          900: "#1e3a8a"
        }
      },
      fontFamily: {
        sans: ["Inter", "system-ui", "sans-serif"]
      }
    }
  },
  plugins: [
    require("@tailwindcss/forms"),
    require("@tailwindcss/typography")
  ]
}
```

### Tailwind v4 (CSS-first config)

Tailwind v4 uses a CSS-based configuration instead of `tailwind.config.js`:

```css
/* app/assets/stylesheets/application.tailwind.css */
@import "tailwindcss";

@source "../../../app/views";
@source "../../../app/javascript";
@source "../../../app/components";

@theme {
  --color-brand: #3b82f6;
  --font-sans: "Inter", system-ui, sans-serif;
}
```

Check which version is installed:

```bash
cat node_modules/tailwindcss/package.json | grep '"version"'
```

### CDN for Prototyping

For prototyping or simple apps with no build step:

```erb
<%# app/views/layouts/application.html.erb %>
<script src="https://cdn.tailwindcss.com"></script>
```

Do not use the CDN in production — it ships the full Tailwind CSS and is slow.

---

## Sass / SCSS

### Setup

```bash
bin/rails css:install:sass
# Installs: sass (Dart Sass)
```

`package.json` build script:

```json
"build:css": "sass ./app/assets/stylesheets/application.scss:./app/assets/builds/application.css --no-source-map --load-path=node_modules"
```

### Entry Point

```scss
// app/assets/stylesheets/application.scss

// Import variables and mixins first
@use "variables" as *;
@use "mixins" as *;

// Import component stylesheets
@use "components/button";
@use "components/card";
@use "components/form";

// Import npm packages (e.g. Bootstrap)
// @use "bootstrap/scss/bootstrap";
```

```scss
// app/assets/stylesheets/_variables.scss
$color-primary: #3b82f6;
$color-danger:  #ef4444;
$font-size-base: 1rem;
$border-radius:  0.375rem;
```

```scss
// app/assets/stylesheets/components/_button.scss
@use "../variables" as *;

.btn {
  display: inline-flex;
  align-items: center;
  padding: 0.5rem 1rem;
  border-radius: $border-radius;
  font-weight: 500;
  transition: background-color 150ms;

  &-primary {
    background-color: $color-primary;
    color: white;

    &:hover {
      background-color: darken($color-primary, 10%);
    }
  }
}
```

### Bootstrap with Sass

```bash
bin/rails css:install:bootstrap
# Installs Bootstrap and Popper.js via npm
```

```scss
// app/assets/stylesheets/application.bootstrap.scss

// Override Bootstrap variables before importing
$primary: #6366f1;
$border-radius: 0.5rem;

@import "bootstrap/scss/bootstrap";
```

---

## PostCSS

### Setup

```bash
bin/rails css:install:postcss
# Installs: postcss, postcss-cli, autoprefixer, postcss-nesting, postcss-import
```

`postcss.config.js`:

```javascript
module.exports = {
  plugins: {
    "postcss-import": {},
    "postcss-nesting": {},
    "autoprefixer": {}
  }
}
```

`package.json` build script:

```json
"build:css": "postcss ./app/assets/stylesheets/application.css -o ./app/assets/builds/application.css"
```

---

## Development: bin/dev

CSS processing requires `bin/dev` (not just `bin/rails server`):

```bash
bin/dev
```

`Procfile.dev` (generated by installer):

```
web: bin/rails server -p 3000
css: yarn build:css --watch
```

When jsbundling is also installed:

```
web: bin/rails server -p 3000
js:  yarn build --watch
css: yarn build:css --watch
```

---

## Production Build

`bin/rails assets:precompile` runs `yarn build:css` automatically:

```bash
RAILS_ENV=production bin/rails assets:precompile
```

This:
1. Runs `yarn build:css`
2. Writes CSS to `app/assets/builds/application.css`
3. Propshaft fingerprints it to `public/assets/application-abc123.css`

Ensure `node` and `yarn` are available on the build server.

---

## Layout Integration

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
</head>
```

`data-turbo-track: "reload"` causes Turbo to do a full page reload if the CSS file changes (e.g. after a new deploy). This prevents users from running stale CSS.

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Styles not updating in dev | CSS watcher not running | Use `bin/dev`, not `bin/rails server` |
| Tailwind classes not generated | `content` path misses a file type | Add the path to `content` in `tailwind.config.js` |
| `Cannot find module 'tailwindcss'` | Node modules not installed | Run `yarn install` |
| Large CSS file in production | Tailwind not purging | Ensure `--minify` flag and correct `content` paths |
| Sass `@import` deprecation warnings | `@import` removed in Dart Sass | Replace with `@use` and `@forward` |
| CSS not loading after deploy | Stale asset reference | Add `data-turbo-track: "reload"` to stylesheet tag |

---

## Related Skills

- `propshaft.md` — serves the compiled CSS output
- `jsbundling.md` — often used alongside cssbundling
- `importmap.md` — JavaScript side (can pair with cssbundling)
- `README.md` — deciding which combination to use
