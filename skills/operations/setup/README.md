# Setup: New Apps, Generators, and Dependencies

This skill covers the first three things you do with Rails: create an application, generate code inside it, and manage its gem dependencies. Use the decision tree to find the right file, then work through it step by step.

## Decision Tree

```
What are you trying to do?
│
├── Create a new Rails application?
│   ├── Full-stack app (HTML, Hotwire, views)?       ──► new-app.md
│   ├── API-only backend?                             ──► new-app.md
│   └── From a template?                              ──► new-app.md
│
├── Generate code inside an existing app?
│   ├── Scaffold a full CRUD resource?               ──► generators.md
│   ├── Model, migration, controller, mailer?        ──► generators.md
│   ├── Background job or Action Cable channel?      ──► generators.md
│   └── Undo a generator you just ran?               ──► generators.md
│
└── Manage gems and dependencies?
    ├── Add, update, or remove a gem?                ──► dependencies.md
    ├── Understand version constraints (~>, >=)?      ──► dependencies.md
    ├── Audit gems for security issues?              ──► dependencies.md
    └── Configure Bundler for CI or Docker?          ──► dependencies.md
```

## Checklist: New App End-to-End

Work through this list in order when starting from scratch:

```
[ ] 1. Check prerequisites
        ruby -v    # 3.2+ for Rails 8, 3.1+ for Rails 7
        rails -v   # 8.x or 7.x

[ ] 2. Generate the app
        rails new myapp --database=postgresql --css=tailwind

[ ] 3. Enter the directory
        cd myapp

[ ] 4. Install dependencies
        bundle install

[ ] 5. Create the database
        bin/rails db:create

[ ] 6. Start the server
        bin/rails server

[ ] 7. Visit http://localhost:3000
        You should see the Rails welcome page.

[ ] 8. Generate your first resource
        bin/rails generate scaffold Article title:string body:text
        bin/rails db:migrate

[ ] 9. Commit the initial state
        git init
        git add .
        git commit -m "Initial Rails app"
```

## Quick Reference

```bash
# New full-stack app
rails new myapp --database=postgresql --css=tailwind

# New API-only app
rails new myapi --api --database=postgresql

# New app skipping components you will replace
rails new myapp --database=postgresql --skip-test --skip-kamal

# Scaffold a complete CRUD resource
bin/rails generate scaffold Article title:string body:text published:boolean
bin/rails db:migrate

# Generate a model only
bin/rails generate model Tag name:string:uniq

# Generate a standalone migration
bin/rails generate migration AddSlugToArticles slug:string:index
bin/rails db:migrate

# Add a gem
echo 'gem "kaminari", "~> 1.2"' >> Gemfile
bundle install

# Update a single gem
bundle update devise

# Check for outdated gems
bundle outdated

# Audit for security vulnerabilities
bundle exec bundle-audit check --update
```

## Rails 7 vs Rails 8: What Changed

Understanding the defaults helps you choose the right flags.

| Concern | Rails 7.x | Rails 8.x |
|---|---|---|
| Asset pipeline | Sprockets (default) or Propshaft | Propshaft |
| JavaScript | Import maps | Import maps |
| Job queue | No default (add Sidekiq or GoodJob) | Solid Queue (DB-backed) |
| Cache store | Memory store | Solid Cache (DB-backed) |
| Action Cable | Async adapter | Solid Cable (DB-backed) |
| Deployment | None built-in | Kamal (Docker-based) |
| Authentication | None built-in | `generate authentication` scaffold |
| Ruby minimum | 2.7.0 | 3.2.0 |
| Primary key type | bigint | bigint |

Rails 8 apps work with SQLite for development **and** production because Solid Queue, Cache, and Cable all use the database rather than Redis. This removes Redis as a required dependency for most apps.

## Skipping Rails 8 Extras You Don't Need

If you deploy to Heroku, Render, or Fly.io without Docker, and already use Redis:

```bash
rails new myapp \
  --database=postgresql \
  --skip-kamal \
  --skip-solid
```

Then add your preferred alternatives to the Gemfile:

```ruby
gem "sidekiq",   "~> 7.3"   # background jobs
gem "redis",     "~> 5.0"   # Redis client
```

## Files in This Skill

| File | What It Covers |
|---|---|
| `new-app.md` | `rails new` flags, Rails 7 vs 8 defaults table, API mode, skip flags, templates, first steps, auth generator |
| `generators.md` | Scaffold, model, controller, migration, mailer, job, channel — field types, modifiers, destroy, dry run, generator config |
| `dependencies.md` | Gemfile structure, groups, version constraints, `bundle install/update`, common gems table, security auditing, Bundler config |

## Prerequisites

Before using this skill, ensure Rails is installed:

```bash
ruby -v    # 3.2.x or later recommended
rails -v   # 8.x or 7.x

# If rails is not installed:
gem install rails
```

If Ruby is not installed, see `skills/foundation/bootstrap/` first.

## Where to Go Next

After creating an app and installing dependencies:

1. **Configure environments and credentials** — `skills/foundation/configuration/`
2. **Understand the directory layout** — `skills/foundation/app-structure/`
3. **Build models, associations, and validations** — `skills/develop/models/`
4. **Write controllers and routes** — `skills/develop/controllers/`
5. **Add views with ERB and Hotwire** — `skills/develop/views/`
