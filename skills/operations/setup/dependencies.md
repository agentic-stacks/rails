# Dependencies: Bundler and Gemfile Management

Bundler resolves, installs, and locks RubyGem dependencies for your Rails application. Every Rails app has a `Gemfile` (declarations) and a `Gemfile.lock` (resolved versions).

## Gemfile Structure

```ruby
# Gemfile
source "https://rubygems.org"

ruby "3.3.6"   # optional but recommended — enforces the Ruby version

gem "rails", "~> 8.0"

# Database
gem "pg", "~> 1.5"

# Asset pipeline
gem "propshaft"

# Development and test only — not loaded in production
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
end

group :development do
  gem "rubocop-rails-omakase", require: false
  gem "web-console"
end

group :test do
  gem "capybara"
  gem "selenium-webdriver"
end

group :production do
  gem "rack-timeout"
end
```

## Version Constraints

| Constraint | Meaning | Example |
|---|---|---|
| `"~> 7.2"` | >= 7.2, < 8.0 (pessimistic, minor) | Allows 7.2.x, not 8.x |
| `"~> 7.2.3"` | >= 7.2.3, < 7.3 (pessimistic, patch) | Allows 7.2.x where x >= 3 |
| `">= 2.0"` | Any version 2.0 or higher | Open upper bound |
| `">= 2.0", "< 3"` | 2.x only | Upper bound set explicitly |
| `"= 1.4.0"` | Exact version only | Pins tightly |
| `"!= 1.3.5"` | Any version except 1.3.5 | Exclude a broken release |
| *(omitted)* | No constraint — any version | Avoid for production |

### Choose the Right Constraint

Use `~> MAJOR.MINOR` for most gems — it follows semantic versioning and accepts compatible bug-fix releases while preventing unexpected major or minor bumps:

```ruby
gem "devise",     "~> 4.9"
gem "pundit",     "~> 2.4"
gem "kaminari",   "~> 1.2"
gem "sidekiq",    "~> 7.3"
gem "pg",         "~> 1.5"
```

Pin exactly (`= x.y.z`) only when a known-breaking release exists and you cannot upgrade yet.

## Core Bundler Commands

```bash
# Install all gems from Gemfile (reads Gemfile.lock if present)
bundle install

# Shorthand
bundle

# Update a single gem to its latest allowed version
bundle update devise

# Update all gems (use with caution in production apps)
bundle update

# Update only gems in a group
bundle update --group=test

# Show all installed gems and their versions
bundle list

# Show where a gem is installed on disk
bundle show sidekiq

# Open a gem's source in your editor
bundle open devise

# Print the gem version
bundle exec gem list devise

# Run a command in the context of the bundle
bundle exec rspec
bundle exec rake db:migrate
```

## Adding a Gem

1. Add the line to `Gemfile`:

```ruby
gem "kaminari", "~> 1.2"
```

2. Install:

```bash
bundle install
```

3. Commit both files:

```bash
git add Gemfile Gemfile.lock
git commit -m "Add kaminari for pagination"
```

## Removing a Gem

1. Delete the line from `Gemfile`.
2. Run:

```bash
bundle install
```

Bundler removes the gem from `Gemfile.lock`. It does not uninstall the gem from your system.

## The Gemfile.lock

Always commit `Gemfile.lock` to version control. It guarantees every developer and CI environment installs identical gem versions.

- Never edit `Gemfile.lock` manually.
- Regenerate it by running `bundle install` or `bundle update`.
- For gems pulled from git, Bundler records the commit SHA in the lock file.

```bash
# Verify the lock file is consistent with the Gemfile
bundle check

# Show why a gem's version was chosen
bundle why sidekiq
```

## Semantic Versioning

Most gems follow `MAJOR.MINOR.PATCH`:

| Increment | Meaning |
|---|---|
| PATCH (`x.y.Z`) | Bug fixes; backward-compatible |
| MINOR (`x.Y.z`) | New features; backward-compatible |
| MAJOR (`X.y.z`) | Breaking changes |

`~> 4.9` expands to `>= 4.9, < 5` — it accepts any 4.x release at or above 4.9, protecting against the breaking 5.0 release.

## Common Gems by Category

| Category | Gem | Notes |
|---|---|---|
| Authentication | `devise` | Full-featured; email/password, OAuth via omniauth |
| Authentication | `rodauth-rails` | More modular; PostgreSQL-first |
| Authentication | *(built-in)* | Rails 8 `generate authentication` for simple cases |
| Authorization | `pundit` | Policy objects; lightweight |
| Authorization | `action_policy` | Policy objects; more expressive than Pundit |
| Pagination | `kaminari` | View helpers + scopes |
| Pagination | `pagy` | Fastest; minimal; no model pollution |
| Search | `ransack` | SQL search and sort from params |
| Search | `pg_search` | Full-text search using PostgreSQL |
| Search | `searchkick` | Elasticsearch-backed search |
| File uploads | `active_storage` | Built-in; local, S3, GCS, Azure |
| File uploads | `shrine` | Plugin-based; fine-grained control |
| Background jobs | `sidekiq` | Redis-backed; high throughput |
| Background jobs | `solid_queue` | DB-backed; Rails 8 default |
| Caching | `solid_cache` | DB-backed; Rails 8 default |
| Caching | `redis-rails` | Redis-backed store |
| API serialization | `blueprinter` | Fast; schema-first |
| API serialization | `alba` | Minimal; fastest benchmarks |
| API serialization | `active_model_serializers` | Older; JSON:API support |
| Admin | `avo` | Modern; resource-based |
| Admin | `administrate` | Convention-driven; customizable |
| Admin | `activeadmin` | Mature; DSL-heavy |
| Monitoring / APM | `sentry-rails` | Error tracking |
| Monitoring / APM | `skylight` | Performance profiling |
| Linting | `rubocop-rails-omakase` | Rails team's RuboCop config |
| Testing | `rspec-rails` | RSpec for Rails |
| Testing | `factory_bot_rails` | Test data factories |
| Testing | `faker` | Fake data generation |

## Gem Sources

Specify alternative sources for private or GitHub-hosted gems:

```ruby
# Private gem server
source "https://gems.example.com" do
  gem "my_private_gem"
end

# Git source (use sparingly — unpinned HEAD is fragile)
gem "active_merchant", git: "https://github.com/activemerchant/active_merchant.git"

# Specific branch
gem "active_merchant", git: "https://github.com/activemerchant/active_merchant.git", branch: "main"

# Specific tag or commit
gem "active_merchant", git: "...", tag: "v1.130.0"
gem "active_merchant", git: "...", ref: "abc1234"

# Local path (development only — do not commit this to shared branches without care)
gem "my_engine", path: "../my_engine"
```

## Security Auditing

`bundler-audit` checks your `Gemfile.lock` against the Ruby Advisory Database.

```bash
# Install
gem install bundler-audit

# Update the advisory database and check
bundle exec bundle-audit check --update

# Check only (skip network update)
bundle exec bundle-audit check
```

Output shows CVE IDs, affected versions, and recommended fix versions. Upgrade flagged gems immediately:

```bash
bundle update <gem_name>
```

Integrate into CI:

```yaml
# .github/workflows/security.yml (GitHub Actions excerpt)
- run: gem install bundler-audit
- run: bundle exec bundle-audit check --update
```

## Bundler Configuration

Bundler reads configuration from `.bundle/config` in the project directory and `~/.bundle/config` globally.

```bash
# Store gems inside the project (useful for Docker / isolated builds)
bundle config set --local path vendor/bundle

# Exclude a group from installation (e.g., skip development gems in CI)
bundle config set --local without development

# Set the number of parallel install jobs
bundle config set --global jobs 4

# Retry on network failures
bundle config set --global retry 3

# Use a specific Gemfile
BUNDLE_GEMFILE=Gemfile.custom bundle install

# Show current config
bundle config list
```

Commit `.bundle/config` only if it contains project-wide settings that every developer needs. Never commit credentials — use environment variables or a credential store instead.

## Keeping Dependencies Healthy

```bash
# Check for outdated gems
bundle outdated

# Check for outdated gems in a specific group
bundle outdated --group=production

# Show gems that have available updates within your version constraint
bundle outdated --strict

# Audit for known vulnerabilities
bundle exec bundle-audit check --update

# Visualise gem dependency graph (requires graphviz)
bundle viz
```

Update production gems in a dedicated PR: one gem at a time, run the full test suite, review the changelog before merging.

## Next Steps

- Generate models and migrations: `generators.md`
- Configure production credentials: `skills/foundation/configuration/`
- Run background jobs with Solid Queue or Sidekiq: `skills/develop/services/`
