# Configuration: Environments, Credentials, and Settings

Rails configuration is layered: framework defaults → `config/application.rb` → `config/environments/*.rb` → initializers. Know which layer to touch for each change.

---

## Environment Configuration

Three environment files control per-environment behavior. They override anything set in `config/application.rb`.

### File Locations

```
config/environments/
├── development.rb   # Local development server
├── production.rb    # Live application
└── test.rb          # Automated test suite
```

### Development Key Settings

```ruby
# config/environments/development.rb
Rails.application.configure do
  # Reload application code between requests
  config.enable_reloading = true

  # Do not eager load — faster boot time in development
  config.eager_load = false

  # Show full exception backtraces in browser
  config.consider_all_requests_local = true

  # Warn in the browser if there are pending migrations
  config.active_record.migration_error = :page_load

  # Raise errors if mailer cannot deliver (not the default)
  config.action_mailer.raise_delivery_errors = false
  config.action_mailer.default_url_options = { host: "localhost", port: 3000 }

  # Use a real caching store if you're testing caching behavior
  # Toggle with: rails dev:cache
  if Rails.root.join("tmp/caching-dev.txt").exist?
    config.cache_store = :memory_store
    config.public_file_server.headers = { "Cache-Control" => "public, max-age=172800" }
  else
    config.action_controller.perform_caching = false
    config.cache_store = :null_store
  end

  # Highlight which templates are being rendered
  config.action_view.annotate_rendered_view_with_filenames = true
end
```

### Production Key Settings

```ruby
# config/environments/production.rb
Rails.application.configure do
  # Eager load all code at boot — required for thread-safe operation
  config.eager_load = true

  # Never show errors to the browser — render public/500.html instead
  config.consider_all_requests_local = false

  # Enforce SSL
  config.force_ssl = true

  # Reduce log noise
  config.log_level = ENV.fetch("RAILS_LOG_LEVEL", "info")

  # Use structured logging with timestamps
  config.log_tags = [:request_id]

  # Set Cache-Control headers on assets served from public/
  config.public_file_server.headers = {
    "Cache-Control" => "public, max-age=#{1.year.to_i}"
  }

  # Store uploaded files in cloud or locally
  config.active_storage.service = :amazon  # matches config/storage.yml key

  # Cache compiled assets
  config.assets.compile = false  # Sprockets only; not used with Propshaft
end
```

### Test Key Settings

```ruby
# config/environments/test.rb
Rails.application.configure do
  config.enable_reloading = false

  # Eager load only when running in CI — catches missing constants before deploy
  config.eager_load = ENV["CI"].present?

  # Raise on all delivery errors so tests catch mailer misconfigurations
  config.action_mailer.delivery_method = :test
  config.action_mailer.raise_delivery_errors = true

  # Raise exceptions instead of rendering error pages
  config.action_dispatch.show_exceptions = :rescuable

  # Disable caching in tests
  config.cache_store = :null_store
end
```

---

## Credentials

Rails encrypts sensitive configuration values and commits the encrypted file. The decryption key is kept out of source control.

### File Layout

```
config/
├── credentials.yml.enc    # Encrypted — SAFE to commit
├── master.key             # Decryption key — NEVER commit (in .gitignore)
└── credentials/           # Per-environment credentials (Rails 6+)
    ├── production.yml.enc
    ├── production.key
    ├── staging.yml.enc
    └── staging.key
```

> WARNING: If you lose `master.key`, the encrypted credentials cannot be recovered. Back it up out of band (password manager, secrets vault).

### Edit Credentials

```bash
# Opens decrypted credentials in $EDITOR
rails credentials:edit

# Edit environment-specific credentials
rails credentials:edit --environment production
rails credentials:edit --environment staging
```

The file is YAML. Structure it however you like:

```yaml
# config/credentials.yml.enc (decrypted view)
secret_key_base: a1b2c3d4e5f6...

aws:
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  region: us-east-1

stripe:
  publishable_key: pk_live_...
  secret_key: sk_live_...

postgres:
  password: supersecret
```

### Access Credentials in Code

```ruby
# Dig into nested keys
Rails.application.credentials.dig(:aws, :access_key_id)
Rails.application.credentials.dig(:stripe, :secret_key)

# Top-level key
Rails.application.credentials.secret_key_base

# With a default (raises KeyError if missing and no default)
Rails.application.credentials.dig(:aws, :region) || "us-east-1"
```

### Per-Environment Credentials

When environment-specific credential files exist, they take precedence over the base `credentials.yml.enc` in that environment.

```bash
# Create production credentials
EDITOR="code --wait" rails credentials:edit --environment production
# Creates: config/credentials/production.yml.enc
#          config/credentials/production.key
```

In production, set `RAILS_MASTER_KEY` from the contents of `production.key`:

```bash
export RAILS_MASTER_KEY=$(cat config/credentials/production.key)
```

---

## Secrets vs Credentials vs ENV vars

Three mechanisms exist. Use the right one for each type of value.

```
What kind of value is it?
│
├── Sensitive, same across all environments?
│   └── Rails credentials (config/credentials.yml.enc)
│
├── Sensitive, different per deployment/server?
│   └── ENV variable set by the deployment platform
│       (Heroku config vars, Kamal .env, Kubernetes secrets)
│
├── Non-sensitive, changes by environment?
│   └── config/environments/*.rb
│
└── Non-sensitive, same across environments?
    └── config/application.rb or config.x custom settings
```

### Rules

| Rule | Reason |
|---|---|
| Never commit `master.key` or `production.key` | Anyone with the key can decrypt all secrets |
| Never commit plain-text passwords or API keys | Rotation is painful; breach is permanent |
| Credentials are safe to commit | They're encrypted with AES-256-GCM |
| ENV vars are for deploy-time overrides | Cloud platforms inject these at runtime |
| Prefer credentials over ENV vars for secrets | Credentials are auditable and versioned |

---

## database.yml

Configures database connections per environment. Supports ERB for dynamic values.

### Default Structure

```yaml
# config/database.yml
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: storage/development.sqlite3

test:
  <<: *default
  database: storage/test.sqlite3

production:
  <<: *default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  database: myapp_production
  username: myapp
  password: <%= Rails.application.credentials.dig(:postgres, :password) %>
  host: <%= ENV["DB_HOST"] %>
```

### DATABASE_URL Override

Setting `DATABASE_URL` in the environment overrides `database.yml` for that environment entirely:

```bash
export DATABASE_URL="postgresql://user:password@localhost/myapp_production"
rails db:migrate  # Uses DATABASE_URL, ignores database.yml for production
```

This is the standard pattern for Heroku, Render, and other PaaS platforms.

### Connection Pooling

```yaml
production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

The pool size should equal `RAILS_MAX_THREADS`. Each Puma thread needs its own database connection. Under-provisioning causes `ActiveRecord::ConnectionTimeoutError`.

```
Connection pool sizing:
  pool size = RAILS_MAX_THREADS (Puma threads per worker)
  
  If you have 2 Puma workers × 5 threads each = 10 connections per process
  Your database max_connections must support: processes × pool_size
```

### Multiple Databases (Rails 6+)

```yaml
# config/database.yml
production:
  primary:
    adapter: postgresql
    database: myapp_primary
    username: myapp
    password: <%= Rails.application.credentials.dig(:postgres, :primary_password) %>
  analytics:
    adapter: postgresql
    database: myapp_analytics
    username: myapp_ro
    password: <%= Rails.application.credentials.dig(:postgres, :analytics_password) %>
    replica: true
```

---

## Custom Configuration

### config.x — Inline Custom Settings

```ruby
# config/application.rb or config/environments/production.rb
config.x.payment_processing.enabled = true
config.x.payment_processing.provider = "stripe"
config.x.features.new_dashboard = false
```

Access anywhere:

```ruby
Rails.configuration.x.payment_processing.provider  # => "stripe"
Rails.configuration.x.features.new_dashboard        # => false
```

### config_for — YAML Config Files

For larger configuration blocks, use a dedicated YAML file under `config/`:

```yaml
# config/payment_processor.yml
default: &default
  currency: usd
  retry_attempts: 3

development:
  <<: *default
  provider: stripe_test
  api_key: pk_test_...

production:
  <<: *default
  provider: stripe
  api_key: <%= Rails.application.credentials.dig(:stripe, :publishable_key) %>
```

```ruby
# config/application.rb
config.payment_processor = config_for(:payment_processor)
```

Access:

```ruby
Rails.configuration.payment_processor[:provider]
Rails.configuration.payment_processor[:currency]
```

---

## Rails 7 vs Rails 8 Configuration Differences

### New Config Files in Rails 8

| File | Purpose |
|---|---|
| `config/deploy.yml` | Kamal deployment settings (servers, image, env vars) |
| `config/queue.yml` | Solid Queue adapter settings (workers, concurrency) |

### config/deploy.yml (Rails 8 / Kamal)

```yaml
# config/deploy.yml
service: myapp
image: myapp/myapp

servers:
  web:
    hosts:
      - 192.168.1.1
    options:
      network: private

registry:
  server: ghcr.io
  username: myuser
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
  clear:
    RAILS_SERVE_STATIC_FILES: "true"
```

### config/queue.yml (Rails 8 / Solid Queue)

```yaml
# config/queue.yml
default: &default
  workers:
    - queues: "*"
      threads: 3
      processes: 1
      polling_interval: 0.1

development:
  <<: *default

production:
  <<: *default
  workers:
    - queues: [default, mailers]
      threads: 5
      processes: 2
```

### Other Rails 7 → 8 Configuration Changes

| Setting | Rails 7 | Rails 8 |
|---|---|---|
| `config.add_autoload_paths_to_load_path` | `true` | `false` — cleaner `$LOAD_PATH` |
| Default job backend | External (Sidekiq recommended) | Solid Queue (DB-backed, built-in) |
| Default cache store | `:memory_store` (dev) | Solid Cache (DB-backed, built-in) |
| Default Action Cable adapter | `async` (dev), Redis (prod) | Solid Cable (DB-backed) |
| Authentication generator | Not built-in | `rails generate authentication` |

---

## Decision Tree: Where to Put Configuration

```
Is this a sensitive value (API key, password, token)?
├── Yes
│   ├── Same value in all environments? → config/credentials.yml.enc
│   └── Different per server/deployment? → ENV variable (set in deploy platform)
└── No
    ├── Changes by environment? → config/environments/<env>.rb
    ├── Small inline setting? → config.x.<namespace>.<setting> in application.rb
    └── Large config block? → config/<name>.yml + config_for(:name)
```

---

## Quick Reference: Common Commands

```bash
# Edit encrypted credentials
rails credentials:edit

# Edit environment-specific credentials
rails credentials:edit --environment production

# Show decrypted credentials (read-only)
rails credentials:show

# Verify RAILS_MASTER_KEY works
RAILS_MASTER_KEY=<key> rails credentials:show

# Check which environment is active
rails runner "puts Rails.env"

# Inspect a config value from the console
rails runner "puts Rails.configuration.x.my_setting"
```

## Next Steps

- Understand the directory layout: `skills/foundation/app-structure/`
- Set up a new app: `skills/operations/setup/`
- Configure the database and run migrations: `skills/develop/models/`
