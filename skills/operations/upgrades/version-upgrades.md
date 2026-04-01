# Version Upgrades

Follow this process for every minor or major Rails upgrade. Repeat the full sequence for each version step — do not batch multiple versions into one branch.

## Step 1: Check the Current Version and Read Release Notes

```bash
# Confirm the current Rails version
bin/rails --version
# => Rails 7.1.5

bundle show rails
```

Read the official release notes for your target version before making any changes:

- Rails 7.1: https://guides.rubyonrails.org/7_1_release_notes.html
- Rails 7.2: https://guides.rubyonrails.org/7_2_release_notes.html
- Rails 8.0: https://guides.rubyonrails.org/8_0_release_notes.html
- Upgrade guide (all versions): https://guides.rubyonrails.org/upgrading_ruby_on_rails.html

Pay attention to: removed APIs, changed defaults, new required configuration, and renamed constants.

## Step 2: Check Known Issues and Gem Compatibility

Check `skills/reference/known-issues/` for any blockers specific to your target version.

Audit your gems for compatibility:

```bash
# Check gems that pin or constrain Rails version
grep -E "rails|activerecord|activesupport|actionpack" Gemfile

# Run a speculative bundle update to see what would change
bundle update --dry-run rails

# Check for gems with known Rails version incompatibilities
bundle outdated --parseable | grep -v "rails"
```

If a critical gem does not yet support the target Rails version, wait for a compatible release before proceeding.

## Step 3: Create a Branch

```bash
git checkout -b upgrade/rails-7-2
```

Name the branch for the target version. If you need multiple version steps, use a separate branch for each.

## Step 4: Update the Gemfile

```ruby
# Gemfile — set the new target version
gem "rails", "~> 7.2.0"
```

For a major upgrade, also update the Ruby version requirement if needed:

```ruby
ruby "3.3.6"   # Rails 8.0 requires Ruby >= 3.2
```

## Step 5: Run bundle update

```bash
# Update Rails and all its dependencies
bundle update rails

# If that pulls in gem conflicts, update specific gems
bundle update rails activerecord activesupport actionpack actionview

# Verify the new version is installed
bundle exec rails --version
# => Rails 7.2.0
```

Resolve any gem conflicts. Prefer updating the conflicting gem over pinning Rails to an older patch.

## Step 6: Run rails app:update

```bash
bin/rails app:update
```

This generator updates initializers, configuration files, and `bin/` scripts for the new version. It is interactive — it will ask whether to overwrite each changed file.

For each file diff it shows:

- Press `d` to see the full diff before deciding.
- Press `Y` to overwrite with the new version (recommended for most config files).
- Press `n` to keep your current version (use when you have custom logic in that file).
- Press `a` to overwrite all remaining files without further prompting.

Review all changes, especially `config/application.rb`, `config/environments/*.rb`, and any new initializers.

See `deprecations.md` for how to handle the `new_framework_defaults_*.rb` file that this generator creates.

## Step 7: Run Database Migrations

Rails itself occasionally ships internal migrations (e.g., for encrypted attributes, new Active Record features):

```bash
bin/rails db:migrate

# Confirm schema is current
bin/rails db:schema:dump   # verify schema.rb updated cleanly
```

## Step 8: Run the Full Test Suite and Fix Deprecations

```bash
# Run everything
bin/rails test

# Or with RSpec
bundle exec rspec
```

Treat every failure as a category:

| Failure type | Action |
|---|---|
| `NameError` / `NoMethodError` on a Rails API | API was removed; check the upgrade guide for the replacement |
| `ArgumentError` / changed method signatures | Check the changelog for the method; update call sites |
| Deprecation warnings (not failures) | Fix now; they become errors in the next version |
| Failing tests due to new default behavior | Read the `new_framework_defaults` file; enable defaults one at a time |

Run the test suite after each fix to catch cascading issues early.

## Step 9: Test Manually, Deploy to Staging, and Merge

```bash
# Start the server in development mode and walk through critical flows
bin/rails server

# Check logs for any runtime deprecations missed by tests
tail -f log/development.log | grep -i deprecat
```

Deploy to a staging environment using the same deployment process as production. Verify:

- Application boots without errors.
- Core user flows work (authentication, primary CRUD operations, background jobs).
- No new exceptions appear in error tracking.

Once staging is stable, open a pull request and merge to main.

---

## Rails 7 to 8 Specific Changes

### Infrastructure: Solid Queue, Solid Cache, Solid Cable

Rails 8 ships three new database-backed adapters as defaults for new apps:

| Component | Rails 8 default | Previous default |
|---|---|---|
| Background jobs | Solid Queue | No default (often Sidekiq/Delayed Job) |
| Caching | Solid Cache | Memory store / Redis |
| Action Cable | Solid Cable | Async adapter / Redis |

For an existing app on Sidekiq or Redis, these are opt-in — your existing adapter configuration is not changed automatically. If you want to adopt them:

```bash
bin/rails solid_queue:install
bin/rails solid_cache:install
bin/rails solid_cable:install
```

Each generates a migration and a configuration file.

### Asset Pipeline: Propshaft

Rails 8 defaults new apps to Propshaft instead of Sprockets. Existing apps are not automatically migrated.

To migrate from Sprockets to Propshaft:

```bash
# Add Propshaft, remove Sprockets
bundle remove sprockets-rails
bundle add propshaft

# Update config/application.rb — remove require "sprockets/railtie"
# Add:  require "propshaft"
```

Propshaft does not support asset preprocessing (Sass, CoffeeScript). If you rely on Sprockets for compilation, migrate to a JS-based build pipeline (cssbundling-rails, jsbundling-rails) first.

### Authentication Generator

Rails 8 ships a built-in authentication generator:

```bash
bin/rails generate authentication
```

This is for new apps. If you have an existing authentication system (Devise, custom), do not run this generator — it will conflict.

### Deployment: Kamal

Rails 8 scaffolds with Kamal for deployment. Existing apps retain their current deployment setup. Kamal is opt-in:

```bash
gem install kamal
kamal init   # generates config/deploy.yml
```

### config.load_defaults 8.0

The most important configuration change when upgrading to Rails 8:

```ruby
# config/application.rb
config.load_defaults 8.0
```

Do not set this all at once on an existing app. Instead, use the `new_framework_defaults_8_0.rb` file generated by `bin/rails app:update` and uncomment settings one at a time. See `deprecations.md`.
