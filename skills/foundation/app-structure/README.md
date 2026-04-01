# App Structure: Rails Directory Layout

Know where everything lives before touching a Rails app. Rails's directory layout is not arbitrary — it encodes the MVC separation, autoloading rules, and deployment conventions.

---

## Annotated Directory Tree

```
my_app/
├── app/                        # Application code — autoloaded by Zeitwerk
│   ├── assets/                 # Stylesheets, images (NOT autoloaded)
│   │   ├── images/
│   │   └── stylesheets/
│   ├── channels/               # Action Cable WebSocket channel classes
│   ├── controllers/            # HTTP request handlers
│   │   ├── application_controller.rb
│   │   └── concerns/           # Shared controller modules
│   ├── helpers/                # View helper methods
│   │   └── application_helper.rb
│   ├── javascript/             # JS entrypoints (NOT autoloaded)
│   ├── jobs/                   # Active Job background job classes
│   │   └── application_job.rb
│   ├── mailers/                # Action Mailer email classes
│   │   └── application_mailer.rb
│   ├── models/                 # Active Record models and POROs
│   │   ├── application_record.rb
│   │   └── concerns/           # Shared model modules
│   └── views/                  # ERB templates (NOT autoloaded)
│       ├── layouts/            # Application layout files
│       │   └── application.html.erb
│       └── <controller>/       # One subdirectory per controller
│
├── bin/                        # Executable scripts
│   ├── dev                     # Starts dev server (Procfile.dev via foreman)
│   ├── rails                   # Rails CLI entry point
│   ├── rake                    # Rake task runner
│   └── setup                   # New developer setup script
│
├── config/                     # Application configuration
│   ├── application.rb          # Core app settings, middleware, module name
│   ├── boot.rb                 # Bundler setup — do not edit
│   ├── credentials.yml.enc     # Encrypted secrets (safe to commit)
│   ├── database.yml            # Database connections per environment
│   ├── deploy.yml              # Kamal deployment config (Rails 8 only)
│   ├── environment.rb          # Initializes the application — do not edit
│   ├── environments/           # Per-environment overrides
│   │   ├── development.rb
│   │   ├── production.rb
│   │   └── test.rb
│   ├── initializers/           # Code run at boot time, alphabetical order
│   ├── locales/                # I18n translation files
│   │   └── en.yml
│   ├── master.key              # Decryption key — NEVER commit
│   ├── puma.rb                 # Puma web server configuration
│   ├── queue.yml               # Solid Queue config (Rails 8 only)
│   └── routes.rb               # URL routing table
│
├── db/                         # Database artifacts
│   ├── migrate/                # Migration files — never edit once run in production
│   ├── schema.rb               # Authoritative current schema (commit this)
│   └── seeds.rb                # Seed data for development / staging
│
├── lib/                        # Non-app Ruby code
│   ├── assets/                 # Assets not specific to app/
│   └── tasks/                  # Custom Rake tasks (*.rake files)
│
├── log/                        # Log files — not committed (in .gitignore)
│   └── development.log
│
├── public/                     # Served directly by web server, no Rails processing
│   ├── 404.html
│   ├── 422.html
│   ├── 500.html
│   └── robots.txt
│
├── storage/                    # Active Storage local file uploads
│
├── test/                       # Minitest test suite (or spec/ for RSpec)
│   ├── controllers/
│   ├── fixtures/               # Test data in YAML
│   ├── helpers/
│   ├── integration/
│   ├── mailers/
│   ├── models/
│   └── system/                 # Capybara system tests
│
├── tmp/                        # Temporary files — not committed
│   ├── cache/
│   ├── pids/
│   └── storage/
│
├── vendor/                     # Vendored gems or JS (rare in modern Rails)
│
├── .ruby-version               # Ruby version declaration for rbenv/asdf/mise
├── Gemfile                     # Gem dependencies
├── Gemfile.lock                # Locked dependency versions — always commit
├── Procfile.dev                # Dev process definitions (web, worker, css)
├── Rakefile                    # Loads Rails rake tasks
└── config.ru                   # Rack entry point (used by Puma)
```

---

## The `app/` Directory in Detail

Zeitwerk autoloads everything under `app/` (except `assets`, `javascript`, and `views`). You never write `require` statements for application code.

### Subdirectories You Add Yourself

Rails does not restrict `app/` to the defaults. Common additions:

| Directory | Contents |
|---|---|
| `app/queries/` | Query objects that encapsulate complex DB queries |
| `app/services/` | Service objects for multi-step business operations |
| `app/serializers/` | Data serializers for JSON API responses |
| `app/presenters/` | Presenter/decorator objects wrapping models for views |
| `app/policies/` | Authorization policy objects (Pundit convention) |
| `app/uploaders/` | CarrierWave uploader classes |

All of these are autoloaded automatically because they live under `app/`.

---

## Autoloading with Zeitwerk

Zeitwerk is the autoloader used in all Rails 7+ apps. It replaces the old `const_missing`-based autoloader. You cannot opt out.

### How Zeitwerk Maps Files to Constants

The rule: **strip the autoload root prefix, then camelize the path**.

```
app/models/user.rb                    → User
app/models/blog_post.rb               → BlogPost
app/controllers/users_controller.rb  → UsersController
app/controllers/admin/users_controller.rb → Admin::UsersController
app/jobs/send_welcome_email_job.rb    → SendWelcomeEmailJob
```

The mapping uses `String#camelize` (ActiveSupport). The autoload root for `app/models/` is `app/models/` — that prefix is stripped, then the remainder is camelized.

### Strict File Naming Rules

```
CORRECT:
  app/models/user.rb          → defines User
  app/models/blog_post.rb     → defines BlogPost

WRONG:
  app/models/User.rb          → Zeitwerk will not load this
  app/models/blogPost.rb      → Zeitwerk will not load this
  app/models/blog-post.rb     → invalid Ruby filename
```

> WARNING: A mismatch between filename and class name causes a `NameError` or `LoadError` at runtime, not at boot in development. This means the error appears only when that constant is first referenced.

### Verifying Your Autoload Setup

```bash
# Check for Zeitwerk violations before deploying
bundle exec rails zeitwerk:check

# Output on success:
# Hold on, I am eager loading the application.
# All is good!
```

Run this in CI. A failure means a file defines a constant that doesn't match its path.

### Configuring Autoload Paths

```ruby
# config/application.rb

# Add a custom directory to autoload paths (rarely needed — app/ subdirs auto-included)
config.autoload_paths << Rails.root.join("lib/my_lib")

# Eager load paths: loaded at boot in production (default: everything in autoload_paths)
# Do not remove paths from eager_load_paths without understanding the impact.
config.eager_load_paths << Rails.root.join("lib/my_lib")
```

### Autoload Once vs Reload

| Autoloader | What it manages | Reloads on change? |
|---|---|---|
| `Rails.autoloaders.main` | Application code under `app/` | Yes, in development |
| `Rails.autoloaders.once` | Code in `autoload_once_paths` | No — loaded once, persists |

Use `autoload_once_paths` for code that is held in long-lived framework objects (e.g., middleware instances) where reloading would leave stale references.

### Reload Behavior in Development

Between requests in development, Rails calls `reload!` which:
1. Unloads all constants from `Rails.autoloaders.main`
2. Next reference to any constant triggers a fresh file load

This means:
- Class definitions are re-evaluated on every request (in development)
- Instance variables on class objects reset
- Objects instantiated before reload are **stale** — they hold a reference to the old class

```
Is your code behaving differently in development vs production?
├── Check if you're storing class-level state (@@variables, class-level memoization)
│   └── These reset on reload in development but persist in production
└── Check if you're comparing class identity (obj.is_a?(User))
    └── Can fail across reloads if the object was created pre-reload
```

---

## Initializers

Files in `config/initializers/` are loaded at application boot, after the framework and gems are loaded but before the first request.

### Execution Order

Initializers run in **alphabetical filename order**. Name them with a numeric prefix when order matters:

```
config/initializers/
├── 01_inflections.rb       # Custom inflection rules (runs first)
├── assets.rb
├── content_security_policy.rb
├── cors.rb
├── filter_parameter_logging.rb
├── permissions_policy.rb
└── sidekiq.rb
```

### Common Initializers

```ruby
# config/initializers/inflections.rb — custom plural/singular rules
ActiveSupport::Inflector.inflections(:en) do |inflect|
  inflect.irregular "octopus", "octopi"
  inflect.uncountable "equipment"
end

# config/initializers/cors.rb — CORS for API-only apps
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV.fetch("CORS_ORIGINS", "localhost:3000")
    resource "*", headers: :any, methods: [:get, :post, :put, :patch, :delete, :options]
  end
end

# config/initializers/filter_parameter_logging.rb — hide sensitive params in logs
Rails.application.config.filter_parameters += [
  :passw, :secret, :token, :_key, :crypt, :salt, :certificate, :otp, :ssn, :cvv, :cvc
]
```

> WARNING: Initializers run once at boot. Code in an initializer that depends on database state (e.g., `User.first`) will fail if the database doesn't exist yet. Use `ActiveSupport.on_load` hooks or lazy evaluation instead.

---

## Environments

Rails ships with three environments. `RAILS_ENV` (or `RACK_ENV`) controls which is active.

```bash
RAILS_ENV=production rails server
RAILS_ENV=test     rails db:migrate
```

| Environment | Purpose | Key behaviors |
|---|---|---|
| `development` | Local coding | Code reloading, detailed errors, no caching, verbose logs |
| `test` | Automated test runs | Transactions wrap each test, no real emails sent, faster boot |
| `production` | Live application | Eager loading, caching on, compressed logs, no detailed errors |

### Per-Environment Config Files

```
config/environments/
├── development.rb   # Overrides for dev
├── production.rb    # Overrides for prod
└── test.rb          # Overrides for test
```

Settings in these files override `config/application.rb` for their environment. Common settings:

```ruby
# config/environments/development.rb
config.enable_reloading = true          # Reload code between requests
config.eager_load = false               # Don't eager load — faster boot
config.consider_all_requests_local = true  # Show full error pages
config.action_mailer.raise_delivery_errors = false  # Don't crash on email errors
config.active_record.migration_error = :page_load  # Warn about pending migrations

# config/environments/production.rb
config.enable_reloading = false
config.eager_load = true                # Load all code at boot (required for thread safety)
config.consider_all_requests_local = false
config.force_ssl = true
config.log_level = :info
config.public_file_server.headers = { "Cache-Control" => "public, max-age=#{1.year.to_i}" }

# config/environments/test.rb
config.enable_reloading = false
config.eager_load = ENV["CI"].present?  # Eager load only in CI
config.action_mailer.delivery_method = :test  # Capture emails in ActionMailer::Base.deliveries
```

### Adding a Custom Environment

```bash
# Create the environment file
cp config/environments/production.rb config/environments/staging.rb

# Use it
RAILS_ENV=staging rails server
```

---

## Key Files Reference

| File | Edit when |
|---|---|
| `config/routes.rb` | Adding or changing URL routes |
| `config/database.yml` | Changing DB adapter, credentials, pool size |
| `config/application.rb` | Adding middleware, configuring framework defaults |
| `config/environments/*.rb` | Environment-specific behavior |
| `config/initializers/*.rb` | Configuring gems at boot |
| `Gemfile` | Adding/removing gem dependencies |
| `db/schema.rb` | Read to understand current DB structure (never edit directly) |
| `db/seeds.rb` | Seed data — run with `rails db:seed` |

## Next Steps

- Configure environments and credentials: `skills/foundation/configuration/`
- Add your first model: `skills/develop/models/`
- Set up routing: `skills/develop/controllers/`
