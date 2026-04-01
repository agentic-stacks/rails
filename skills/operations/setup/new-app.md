# Create a New Rails Application

`rails new` generates a complete application skeleton. Run it outside any existing Rails app.

## Full-Stack App (Rails 8 Defaults)

```bash
rails new myapp
cd myapp
bin/rails db:create
bin/rails server
# visit http://localhost:3000
```

Rails 8 generates with these defaults out of the box:

| Concern | Rails 8 Default | Rails 7.1 Default |
|---|---|---|
| Database | SQLite3 | SQLite3 |
| Asset pipeline | Propshaft | Sprockets |
| JavaScript | Import maps | Import maps |
| CSS | No framework (plain CSS) | No framework |
| Job queue | Solid Queue | No default |
| Cache | Solid Cache | Memory store |
| WebSockets/Cable | Solid Cable | Action Cable (async) |
| Deployment | Kamal | None built-in |
| Auth generator | `bin/rails generate authentication` | None built-in |
| Ruby minimum | 3.2.0 | 2.7.0 |

## Common Option Combinations

### PostgreSQL + Tailwind + esbuild

```bash
rails new myapp \
  --database=postgresql \
  --css=tailwind \
  --javascript=esbuild
```

This installs:
- `pg` gem and configures `config/database.yml` for PostgreSQL
- `tailwindcss-rails` gem with a `tailwind.config.js`
- `jsbundling-rails` + esbuild for JavaScript bundling (requires Node)

### PostgreSQL + Tailwind (no Node required)

```bash
rails new myapp \
  --database=postgresql \
  --css=tailwind
```

Uses `tailwindcss-rails` standalone binary — no Node or npm needed.

### MySQL

```bash
rails new myapp --database=mysql
```

### API-Only App

```bash
rails new myapi --api --database=postgresql
```

`--api` removes the view layer, session middleware, and cookie middleware. The generated `ApplicationController` inherits from `ActionController::API` instead of `ActionController::Base`.

## Database Options

| Flag | Adapter gem |
|---|---|
| `--database=sqlite3` | `sqlite3` (default) |
| `--database=postgresql` | `pg` |
| `--database=mysql` | `mysql2` |
| `--database=trilogy` | `trilogy` (MySQL protocol, no libmysql) |

## CSS Options

| Flag | What it installs |
|---|---|
| *(omitted)* | Plain CSS via Propshaft |
| `--css=tailwind` | `tailwindcss-rails` standalone |
| `--css=bootstrap` | `cssbundling-rails` + Bootstrap via npm |
| `--css=bulma` | `cssbundling-rails` + Bulma via npm |
| `--css=sass` | `cssbundling-rails` + Sass via npm |

## JavaScript Options

| Flag | What it installs |
|---|---|
| *(omitted)* | Import maps (no Node required) |
| `--javascript=esbuild` | `jsbundling-rails` + esbuild |
| `--javascript=webpack` | `jsbundling-rails` + webpack |
| `--javascript=rollup` | `jsbundling-rails` + rollup |

## Skip Flags

Use skip flags to remove components you will replace or do not need:

```bash
rails new myapp \
  --skip-action-mailer \   # no Action Mailer (add a third-party mailer later)
  --skip-test \            # skip Minitest (use --skip-test when adding RSpec)
  --skip-kamal \           # skip Kamal deployment config (Rails 8 default includes it)
  --skip-solid \           # skip Solid Queue/Cache/Cable (use Redis-backed alternatives)
  --skip-hotwire \         # skip Turbo and Stimulus
  --skip-jbuilder          # skip jbuilder views
```

Common combination for an RSpec-based app without Kamal:

```bash
rails new myapp \
  --database=postgresql \
  --skip-test \
  --skip-kamal
```

Then add RSpec:

```ruby
# Gemfile
group :development, :test do
  gem "rspec-rails"
end
```

```bash
bundle install
bin/rails generate rspec:install
```

## App Templates

Apply a template file during creation with `-m`:

```bash
rails new myapp -m template.rb
rails new myapp -m https://example.com/my_template.rb
```

A template is a Ruby script using the Thor DSL:

```ruby
# template.rb
gem "devise"
gem "pundit"

after_bundle do
  generate "devise:install"
  rails_command "db:create db:migrate"
  git :init
  git add: "."
  git commit: "-m 'Initial commit'"
end
```

Popular community templates:
- `https://railsbytes.com` — browsable collection of community templates
- `jumpstart` (gorails) — full-featured SaaS starter

## After Creation: First Steps

```bash
# 1. Enter the app directory
cd myapp

# 2. Create the database (SQLite: also creates the file)
bin/rails db:create

# 3. Start the development server
bin/rails server
# or shorthand:
bin/rails s

# 4. Visit the app
open http://localhost:3000

# 5. Open the Rails console
bin/rails console
# or shorthand:
bin/rails c
```

For PostgreSQL, ensure the database server is running before `db:create`. With the `pg` gem, `database.yml` defaults to connecting as the current OS user with no password — adjust host/username/password if your Postgres install uses authentication.

## Rails 8: Built-in Authentication Generator

Rails 8 ships an authentication generator that creates a session-based auth system:

```bash
bin/rails generate authentication
bin/rails db:migrate
```

This generates:
- `app/models/user.rb` — with `has_secure_password` and session token logic
- `app/models/session.rb` — persisted session record
- `app/controllers/sessions_controller.rb` — login/logout
- `app/controllers/passwords_controller.rb` — password reset
- Migrations for `users` and `sessions` tables
- `before_action :require_authentication` helper in `ApplicationController`

This is a starting point. For OAuth, multi-factor auth, or advanced features use Devise or Rodauth.

## Verify the Generated App

```bash
# Check Rails and Ruby versions the app expects
cat .ruby-version          # Ruby version (if generated)
grep '"rails"' Gemfile     # Rails version pin

# List all npm scripts (if using jsbundling/cssbundling)
cat package.json

# Check the database config
cat config/database.yml
```

## Next Steps

- Configure credentials and environments: `skills/foundation/configuration/`
- Generate models, controllers, and migrations: `generators.md`
- Add and manage gems: `dependencies.md`
