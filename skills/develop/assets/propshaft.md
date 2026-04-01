# Propshaft: Asset Pipeline for Rails 8

Propshaft is the default asset pipeline in Rails 8. It replaces Sprockets with a simpler model: find assets, fingerprint them, serve them. No compilation, no concatenation, no preprocessing.

## What Propshaft Does

- Finds all files in configured asset load paths
- Fingerprints them with a content hash (`logo-abc123def456.png`)
- Writes them to `public/assets/` on precompile
- Generates a manifest (`assets/.manifest.json`)
- Provides `asset_path`, `asset_url` helpers and `stylesheet_link_tag` / `javascript_include_tag`

## What Propshaft Does NOT Do

- No ERB processing in asset files
- No Sass/Less compilation (use cssbundling for that)
- No JavaScript transpilation or bundling (use jsbundling for that)
- No concatenation of multiple files into one

---

## Installation

### New App (Rails 8)

Propshaft is the default. Nothing extra needed:

```bash
rails new myapp
```

### New App (Rails 7) — Opt In

```bash
rails new myapp --asset-pipeline propshaft
```

### Add to Existing App

```bash
bundle remove sprockets sprockets-rails
bundle add propshaft
bin/rails propshaft:install
```

`propshaft:install` adds a `Propshaft::Engine` railtie entry and cleans up Sprockets configuration.

---

## Gemfile

```ruby
gem "propshaft"   # Included by default in Rails 8
```

No additional gems needed for basic usage.

---

## Asset Load Paths

Propshaft scans these directories by default:

```
app/assets/
lib/assets/
vendor/assets/
```

All subdirectories are included. Gem assets are added automatically.

### Add a Custom Path

```ruby
# config/initializers/assets.rb (or config/application.rb)
config.assets.paths << Rails.root.join("node_modules/@fontsource/inter/files")
```

### Sweeping the Load Path

Check what Propshaft sees:

```bash
bin/rails assets:environment
# or check the manifest after precompile:
cat public/assets/.manifest.json | python3 -m json.tool | head -50
```

---

## Using Assets in Views

```erb
<%# Fingerprinted URL %>
<%= image_tag "logo.png" %>
<%# => <img src="/assets/logo-abc123.png"> %>

<%= stylesheet_link_tag "application" %>
<%# => <link rel="stylesheet" href="/assets/application-def456.css"> %>

<%# Raw path (for use in CSS or JS) %>
<%= asset_path("logo.png") %>
<%# => "/assets/logo-abc123.png" %>

<%# Full URL (for emails, meta tags) %>
<%= asset_url("logo.png") %>
<%# => "https://myapp.com/assets/logo-abc123.png" %>
```

### Referencing Assets from CSS

Propshaft does not process CSS. Use absolute paths or rely on your CSS bundler to resolve references:

```css
/* Works — direct path, but not fingerprinted in plain CSS */
background-image: url("/assets/hero.jpg");

/* With cssbundling (PostCSS/Sass), asset references are resolved by the bundler */
background-image: url("../images/hero.jpg");
```

If you need fingerprinted asset URLs inside CSS, use an ERB stylesheet (only supported with Sprockets). With Propshaft, put images in `public/` and reference them directly, or use inline styles with `asset_path` in views.

---

## Directory Structure

```
app/
  assets/
    images/
      logo.png
      icons/
        arrow.svg
    stylesheets/
      application.css     ← entry point imported by layout
      components/
        button.css
    javascripts/           ← rarely used with Import Maps; JS lives in app/javascript/
```

With Import Maps, JavaScript files live in `app/javascript/` and are served by Rails's Rack middleware directly (not Propshaft). With jsbundling, the build output (`app/assets/builds/`) is added to Propshaft's path.

---

## Digest / Fingerprinting

Propshaft computes a SHA1 digest of the file content and embeds it in the filename:

```
logo.png  →  logo-6ef33a97bc.png
```

The manifest maps logical names to fingerprinted names:

```json
{
  "logo.png": "logo-6ef33a97bc.png",
  "application.css": "application-3a2b1c.css"
}
```

`asset_path("logo.png")` looks up the manifest and returns the fingerprinted path. In development, the manifest is rebuilt on every request so changes are picked up immediately.

---

## Precompile

```bash
# Compile and fingerprint all assets to public/assets/
RAILS_ENV=production bin/rails assets:precompile

# Clean old fingerprinted files (keep last 2 versions)
RAILS_ENV=production bin/rails assets:clean
RAILS_ENV=production bin/rails assets:clean[2]
```

Typically run as part of the deploy process:

```yaml
# Example Kamal deploy hook or CI step
- bundle exec rails assets:precompile
```

### Skip Fingerprinting for a File

Propshaft fingerprints everything by default. To serve a file at a stable path (e.g., a manifest used by a PWA), put it in `public/` instead of `app/assets/`.

---

## Differences from Sprockets

| Capability | Sprockets | Propshaft |
|---|---|---|
| ERB in `.js.erb` or `.css.erb` asset files | Yes | No |
| Sass/SCSS compilation | Built-in (`sass-rails`) | No — use cssbundling |
| `//= require` directives | Yes | No |
| Asset concatenation | Yes | No |
| Sourcemaps | Partial | Passes through if bundler generates them |
| Complexity | High | Low |
| Config surface area | Large | Minimal |
| Rails 8 default | No | Yes |

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `ActionView::Template::Error: Asset not found` | File not in load path | Check `config.assets.paths`; verify filename spelling |
| Asset not fingerprinted in production | Missing `asset_path` / `asset_tag` helper | Use `asset_path` instead of hardcoded `/assets/` paths |
| `//= require` directives not working | Propshaft ignores Sprockets directives | Concatenate CSS manually or use a bundler |
| ERB in `.css.erb` not rendered | Propshaft does not process ERB in assets | Move ERB logic to views, inline styles, or CSS custom properties |
| Old asset still cached | Stale manifest | Run `assets:clean` then `assets:precompile` |

---

## Related Skills

- `importmap.md` — JavaScript with Import Maps (no bundler)
- `jsbundling.md` — JavaScript with esbuild/rollup/webpack
- `cssbundling.md` — Tailwind, Sass, PostCSS
- `README.md` — choosing the right asset pipeline combination
