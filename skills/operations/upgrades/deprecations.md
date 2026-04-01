# Handling Deprecations

A deprecation warning means: this API works today, but will be removed in the next version. Fix every deprecation in the current version before upgrading. Deferring them means encountering hard errors instead of guided warnings.

## Deprecation Timeline

| Event | Version |
|---|---|
| Warning first appears | N (current) |
| API removed, warning gone | N+1 |

Example: A method deprecated in Rails 7.1 raises `NoMethodError` in Rails 7.2. Fix warnings in 7.1 while you still have the deprecation message to tell you what to use instead.

## Finding Deprecation Warnings

### In Tests (Recommended Starting Point)

Raise deprecation warnings as errors so failing tests catch them before production does:

```ruby
# config/environments/test.rb
config.active_support.deprecation = :raise
```

With this set, any deprecated API call in a test causes the test to fail with a `ActiveSupport::DeprecationException`. Run the full suite:

```bash
bin/rails test

# Or with RSpec
bundle exec rspec
```

Fix each failure. The exception message names the deprecated method and the replacement.

Remove the `:raise` config once all warnings are resolved, or keep it permanently to prevent new deprecation debt from accumulating.

### In Development Logs

```bash
# Start the server and filter for deprecations
bin/rails server
tail -f log/development.log | grep -i deprecat

# Or search the log file after a session
grep -i "deprecat" log/development.log | sort -u
```

### In the Rails Console

```bash
bin/rails console

# Turn on verbose deprecation output for your session
ActiveSupport::Deprecation.behavior = :raise

# Then exercise the code path you suspect
User.find(1).some_deprecated_method
```

### Searching the Codebase

If you know the specific deprecated method (from the upgrade guide or a warning message), find all call sites:

```bash
# Find all uses of a deprecated method
grep -rn "method_name" app/ lib/ config/

# Find uses of a deprecated constant
grep -rn "OldConstantName" app/ lib/
```

## rails app:update

`bin/rails app:update` is the official tool for adopting new configuration introduced with a Rails version. Run it every time you upgrade.

```bash
bin/rails app:update
```

The generator is interactive. For each file it wants to change:

```
Overwrite config/application.rb? (enter "h" for help) [Ynaqdhm]
```

| Key | Action |
|---|---|
| `Y` | Overwrite with the new Rails version |
| `n` | Keep your current file |
| `d` | Show the diff before deciding |
| `a` | Overwrite this and all remaining files |
| `h` | Show the help menu |

**Strategy:** Press `d` first on every file. For files you have not customized (e.g., `bin/rails`, `bin/setup`), overwrite. For files with custom logic (e.g., `config/application.rb`, `config/environments/production.rb`), compare carefully and merge manually.

### What app:update Generates

Beyond updating existing files, the generator creates:

- `config/initializers/new_framework_defaults_X_Y.rb` — new defaults file (see below)
- Updated `config/environments/*.rb` with new options
- Updated `bin/` executables to the new Rails version

## New Framework Defaults

When you upgrade Rails, `bin/rails app:update` generates a file like:

```
config/initializers/new_framework_defaults_8_0.rb
```

This file contains the new behavior flags introduced in 8.0, all commented out. The flags are already active in new Rails 8.0 apps (via `config.load_defaults 8.0`), but existing apps get them as an opt-in migration path.

**Do not uncomment everything at once.** Enable settings one at a time, run the full test suite after each, and fix any failures before moving to the next setting.

```ruby
# config/initializers/new_framework_defaults_8_0.rb

# Uncomment ONE setting at a time:

# Rails.application.config.active_support.cache_format_version = 7.1
# Rails.application.config.action_dispatch.default_headers = { ... }
# Rails.application.config.active_record.run_commit_callbacks_on_first_saved_instances = false
```

### Workflow

```bash
# 1. Open the defaults file
$EDITOR config/initializers/new_framework_defaults_8_0.rb

# 2. Uncomment one setting

# 3. Run the test suite
bin/rails test

# 4. Fix any failures caused by the new behavior

# 5. Commit the enabled setting before moving to the next one
git add config/initializers/new_framework_defaults_8_0.rb
git commit -m "Enable active_record.run_commit_callbacks_on_first_saved_instances default"

# 6. Repeat for the next setting
```

Once all settings are enabled, update `config/application.rb` to use the new load defaults directly and delete the initializer:

```ruby
# config/application.rb
config.load_defaults 8.0   # replaces the separate initializer
```

```bash
rm config/initializers/new_framework_defaults_8_0.rb
git add -A
git commit -m "Switch to config.load_defaults 8.0, remove defaults initializer"
```

## Common Deprecation Patterns

### Method renamed or moved

```
DEPRECATION WARNING: `foo` is deprecated and will be removed in Rails 8.0.
Use `bar` instead.
```

Find all call sites and replace `foo` with `bar`.

### Keyword argument changes

Ruby 3.0 separated positional and keyword arguments. Rails APIs updated accordingly:

```ruby
# Deprecated (Rails 7.x warning, errors in some contexts)
SomeClass.new({ key: value })

# Correct
SomeClass.new(key: value)
```

### ActiveRecord finder changes

```ruby
# Deprecated
User.find_all_by_name("Alice")     # dynamic finders removed in Rails 4 — check older apps

# Current
User.where(name: "Alice")
User.find_by(name: "Alice")        # returns first match or nil
User.find_by!(name: "Alice")       # raises ActiveRecord::RecordNotFound if missing
```

### Silencing vs. Raising

Use `:raise` in test, `:log` in development, and `:notify` (with an error tracker) in production if you want runtime coverage:

```ruby
# config/environments/development.rb
config.active_support.deprecation = :log

# config/environments/test.rb
config.active_support.deprecation = :raise

# config/environments/production.rb
config.active_support.report_deprecations = false   # Rails 7.1+ syntax
```

In Rails 7.1+, `ActiveSupport::Deprecation` became an instance-based object. Check the upgrade guide for the current configuration API if the above options do not apply to your version.
