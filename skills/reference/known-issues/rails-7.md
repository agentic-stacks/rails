# Rails 7 Known Issues

Known bugs and workarounds for Rails 7.0, 7.1, and 7.2.

---

## Rails 7.0

### Zeitwerk Strict Mode Rejects Files That Were Valid Under Classic

**Symptom:** `LoadError: expected .../app/models/my_file.rb to define constant MyFile` on boot after upgrading from Rails 6.

**Cause:** Rails 7 uses Zeitwerk as the only supported autoloader. Zeitwerk enforces a strict mapping between file names and constant names. Any file that was silently tolerated by the classic autoloader — misnamed constants, `require` calls for app files, non-standard directory structures — now raises at boot.

**Affected versions:** 7.0.0+

**Status:** `workaround only` — strict mode is intentional; files must be renamed or restructured.

**Workaround:**
1. Run the upgrade check task:
   ```bash
   bin/rails zeitwerk:check
   ```
2. Rename files so the constant defined inside matches the file name exactly (snake_case file → CamelCase constant).
3. Remove any `require` or `require_dependency` calls for files under `app/`. Zeitwerk handles autoloading.
4. For third-party files that cannot be renamed, place them outside `app/` and `require` them explicitly in an initializer.

---

### Sprockets 4 Digest Fingerprinting Breaks asset_path in Emails

**Symptom:** `ActionView::Template::Error: The asset "image.png" is not present in the asset pipeline` when rendering mailer views that reference images via `asset_path` or `image_tag` in production.

**Cause:** Sprockets 4 changed how it handles asset digests in non-request contexts (like mailers). The manifest lookup fails when `config.assets.digest` is `true` and the manifest file is not accessible to the mailer rendering context.

**Affected versions:** 7.0.0–7.0.4 with Sprockets 4.x

**Status:** `fixed in 7.0.5` — also addressed by Sprockets 4.2.

**Workaround:**
```ruby
# config/environments/production.rb
# Ensure the manifest path is explicitly set:
config.assets.manifest = Rails.root.join("public/assets/.sprockets-manifest-*.json")
```
Or upgrade to Sprockets `>= 4.2.0`.

---

### Dockerfile Generator Produces Non-Buildable Image for Apps with Native Gems

**Symptom:** `docker build` fails with errors like `make: not found` or missing `-dev` packages when the generated Dockerfile is used with gems that require native extensions (e.g., `pg`, `nokogiri`, `sassc`).

**Cause:** The initial `rails new --dockerfile` generator (shipped in 7.0) did not install build tools (`build-essential`, `libpq-dev`, etc.) in the build stage. It assumed pure-Ruby gems only.

**Affected versions:** 7.0.0–7.0.3

**Status:** `fixed in 7.0.4`

**Workaround:**
Add build dependencies to the Dockerfile build stage:
```dockerfile
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      build-essential \
      libpq-dev \
      libvips && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives
```

---

### `config.hosts` Blocks Requests in Development After Upgrade

**Symptom:** `BlockedHost` error page in development after upgrading, even for `localhost`.

**Cause:** Rails 7 enforces `config.hosts` more strictly. If your `development.rb` does not explicitly allow your local hostname (e.g., when using a `.local` domain or Docker hostname), requests are blocked.

**Affected versions:** 7.0.0+

**Status:** `workaround only`

**Workaround:**
```ruby
# config/environments/development.rb
config.hosts << "myapp.local"
# Or to allow all hosts in dev (not recommended for CI):
config.hosts.clear
```

---

## Rails 7.1

### `ActiveRecord::Base.logger` Silenced by Default in Fixtures Load

**Symptom:** Zero SQL logging during `rails db:fixtures:load` even when `Rails.logger` is set to debug level.

**Cause:** Rails 7.1 added automatic logger silencing during bulk fixture load to reduce noise. The silence scope is broader than intended and suppresses user-configured loggers in addition to the default Rails logger.

**Affected versions:** 7.1.0–7.1.2

**Status:** `fixed in 7.1.3`

**Workaround:**
```ruby
# config/environments/development.rb
config.active_record.verbose_query_logs = true
```

---

### Bun Not Recognised by `./bin/dev` When Used as JavaScript Runtime

**Symptom:** `bundle exec foreman start` exits immediately or falls back to Node when Bun is installed but Rails' `Procfile.dev` still references `yarn` or `node`.

**Cause:** Rails 7.1 added first-class Bun support via `--javascript=bun` but the `bin/dev` and `Procfile.dev` templates were not updated consistently across patch releases. Existing apps that manually swap to Bun do not have the correct Procfile entries.

**Affected versions:** 7.1.0–7.1.1

**Status:** `fixed in 7.1.2`

**Workaround:**
Update `Procfile.dev` manually:
```
web: bin/rails server
js: bun run build --watch
css: bin/rails tailwindcss:watch   # if using Tailwind
```
And update `package.json` scripts to use `bun` instead of `yarn`.

---

### `assert_emails` Matcher Fails When Using Async Mailer Delivery

**Symptom:** `assert_emails 1` passes 0 even though the mailer was called, because the job has not been processed yet.

**Cause:** Rails 7.1 added `config.action_mailer.delivery_job` support for async delivery. When the test queue adapter is used, `assert_emails` does not automatically flush the job queue before counting.

**Affected versions:** 7.1.0+

**Status:** `workaround only`

**Workaround:**
```ruby
# Flush jobs before asserting
perform_enqueued_jobs do
  post users_path, params: { user: valid_attributes }
end
assert_emails 1
```
Or configure delivery mode to `:inline` in the test environment:
```ruby
# config/environments/test.rb
config.action_mailer.delivery_method = :test
config.action_mailer.perform_deliveries = true
```

---

### Propshaft + importmap: `bin/importmap` Fails to Pin Vendored Assets

**Symptom:** `bin/importmap pin react` succeeds but the pinned path is wrong when Propshaft is the asset pipeline (instead of Sprockets).

**Cause:** The importmap-rails gem assumed Sprockets' public directory layout when constructing vendored asset paths. Propshaft uses a different output directory structure.

**Affected versions:** importmap-rails `< 2.0` with Propshaft

**Status:** `fixed in importmap-rails 2.0`

**Workaround:**
Upgrade `importmap-rails` to `>= 2.0`:
```ruby
# Gemfile
gem "importmap-rails", ">= 2.0"
```

---

## Rails 7.2

### `bin/rails generate dockerfile` Omits `RUBY_VERSION` ARG When Using `.ruby-version`

**Symptom:** Generated Dockerfile hard-codes the Ruby version string instead of reading from `.ruby-version`, causing image rebuilds to go stale after a Ruby patch upgrade.

**Cause:** The Dockerfile generator in 7.2 reads `RUBY_VERSION` from the environment or the Gemfile lock at generation time and bakes it in as a literal. It does not emit an ARG or copy `.ruby-version`.

**Affected versions:** 7.2.0–7.2.1

**Status:** `workaround only`

**Workaround:**
Add to the top of the generated Dockerfile:
```dockerfile
ARG RUBY_VERSION=3.3.0
FROM ruby:${RUBY_VERSION}-slim AS base
```
Then pass `--build-arg RUBY_VERSION=$(cat .ruby-version)` in your build command or CI pipeline.

---

### Solid Cache Eviction Causes Lock Contention on SQLite in Development

**Symptom:** Slow requests or `SQLite3::BusyException` errors in development when Solid Cache eviction runs alongside web requests.

**Cause:** Solid Cache's background eviction thread in Rails 7.2 uses a single write connection. SQLite does not support concurrent writers; if the eviction thread holds the write lock during a web request that also writes to SQLite, requests block.

**Affected versions:** 7.2.0 with SQLite adapter

**Status:** `fixed in 7.2.2` — eviction now runs less aggressively and yields on lock contention.

**Workaround:**
Increase SQLite busy timeout in `config/database.yml`:
```yaml
development:
  adapter: sqlite3
  database: storage/development.sqlite3
  timeout: 5000
```
Or disable Solid Cache background eviction in development:
```ruby
# config/environments/development.rb
config.cache_store = :solid_cache_store, { expiry_method: :null_expiry }
```
