# Rails Upgrades

Upgrading Rails requires systematic preparation, incremental version steps, and validation at each stage. Rushing an upgrade — or skipping versions — causes compounding breakage that is difficult to untangle.

## Core Rules

**Upgrade one minor version at a time.** Never jump from 7.0 to 8.0 directly.

```
7.0 → 7.1 → 7.2 → 8.0 → 8.1
```

Each version deprecates features that the next version removes. Skipping a step means encountering removals without deprecation warnings to guide you.

**Check known issues before starting.** Read `skills/reference/known-issues/` for any version-specific problems or gem compatibility blockers that apply to your app.

**Work on a dedicated branch.** Never upgrade on `main`. Upgrades require iteration.

## Time Budget

| Upgrade type | Typical range | What drives the upper bound |
|---|---|---|
| Minor (e.g., 7.1 → 7.2) | 1–4 hours | Test suite size, gem compatibility |
| Major (e.g., 7.2 → 8.0) | 1–3 days | New defaults, removed APIs, infrastructure changes |

These assume a healthy test suite. If coverage is low, add time for manual verification.

## Decision Tree: Which Skill to Read

```
What do you need to do?
│
├── Upgrade Rails to the next version?
│   └── version-upgrades.md
│
├── Fix deprecation warnings in the current version?
│   └── deprecations.md
│
├── Run rails app:update to adopt new config files?
│   └── deprecations.md  (see "rails app:update" section)
│
└── Adopt new framework defaults from a new_framework_defaults file?
    └── deprecations.md  (see "New Framework Defaults" section)
```

## Files in This Skill

| File | What it covers |
|---|---|
| `version-upgrades.md` | Step-by-step Rails version upgrade process, Rails 7→8 specifics |
| `deprecations.md` | Finding and fixing deprecation warnings, rails app:update, new defaults |

## Before You Start Any Upgrade

1. Read the official Rails upgrade guide for your target version:
   `https://guides.rubyonrails.org/upgrading_ruby_on_rails.html`
2. Check `skills/reference/known-issues/` for this version.
3. Confirm your Ruby version satisfies the new Rails requirement.
4. Ensure the test suite is green on the current version before touching anything.

| Rails Version | Minimum Ruby | Recommended Ruby |
|---|---|---|
| Rails 7.0 | Ruby 2.7 | Ruby 3.2.x |
| Rails 7.1 | Ruby 2.7 | Ruby 3.2.x |
| Rails 7.2 | Ruby 3.1 | Ruby 3.3.x |
| Rails 8.0 | Ruby 3.2 | Ruby 3.3.x |

A test suite that was green before the upgrade is your baseline. Every failure after the upgrade points to a specific incompatibility — that is the goal of incremental upgrading.

## Upgrade Path for Rails 7.x and 8.x

### 7.0 → 7.1

Key changes: `ActiveRecord::Base.generate_secure_token`, `config.active_record.query_log_tags_format`, new `ActiveSupport::MessageEncryptor` interface, async queries. The upgrade guide lists several renamed configuration keys.

Estimated time: 1–2 hours for a well-tested app.

### 7.1 → 7.2

Key changes: Ruby 3.1 becomes the minimum requirement. Rails DevContainers support, new browser version guard (`config.browser_versions_to_support`), Puma as required server for development. Sprockets is still supported but Propshaft is available as an alternative.

Estimated time: 1–3 hours. Most effort is usually dependency compatibility.

### 7.2 → 8.0

Key changes: Solid Queue, Solid Cache, Solid Cable as default adapters for new apps (existing apps opt in). Authentication generator. Kamal in default scaffolding. `config.load_defaults 8.0` introduces behavior changes in Active Record callbacks and Active Support cache formats. Propshaft becomes the default for new apps.

Estimated time: 1–3 days. The new infrastructure components and framework defaults need careful, incremental adoption.

## Validating the Upgrade

After every version step, before proceeding to the next:

```bash
# Test suite must be fully green
bin/rails test

# No deprecation warnings in test output
# (set config.active_support.deprecation = :raise in test.rb first)

# Application boots cleanly in all environments
RAILS_ENV=production bin/rails runner "puts Rails.version"

# Database is in sync
bin/rails db:migrate:status   # all should show "up"
```

Do not start the next version step until all four checks pass. A partially upgraded app is harder to debug than one at a known stable state.

## Gem Compatibility Checklist

Before upgrading, check each of these gem categories:

| Category | Check |
|---|---|
| Authentication (Devise, Rodauth) | Verify it supports the target Rails version |
| Admin (ActiveAdmin, RailsAdmin) | Slower to update; may need a beta/main version |
| API layer (Grape, Jsonapi-rb) | Check changelogs for compatibility notes |
| Testing (FactoryBot, Shoulda) | Usually well-maintained, but verify |
| Asset pipeline (Sprockets) | Version must match Rails expectations |
| Background jobs (Sidekiq, Delayed Job) | Check Rails adapter compatibility |

Pin gems that do not yet support the target version and upgrade them separately once compatible releases are available.
