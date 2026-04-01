# Boot Failures

Use this file when your Rails application crashes before it can serve any request. Work through the decision tree from top to bottom — each branch ends with a verification step.

## Quick Triage

```bash
# Run the app and capture the full error
bin/rails server 2>&1 | head -60

# Or boot without the server to isolate the load error
bin/rails runner "puts 'boot OK'"
```

The first line of the error message tells you which branch to follow below.

---

## Decision Tree

```
App crashes on boot
  │
  ├─ 1. Ruby version error ("Your Ruby version is X, but your Gemfile specified Y")
  ├─ 2. Bundler / gem errors ("Could not find gem", "LoadError")
  ├─ 3. Database connection error ("ConnectionNotEstablished", "NoDatabaseError")
  ├─ 4. Missing environment variable (KeyError / RuntimeError in initializer)
  ├─ 5. Initializer error (exception raised in config/initializers/)
  ├─ 6. Asset compilation error (ExecJS::RuntimeError, Sprockets::FileNotFound)
  └─ 7. Spring issues (stale process, wrong code loaded) — Rails 7
```

---

## 1. Ruby Version Error

**Symptom:**
```
Your Ruby version is 3.1.4, but your Gemfile specified ruby "3.2.2"
```

### Check

```bash
ruby --version
cat .ruby-version
cat Gemfile | grep "^ruby"
```

### Fix

```bash
# rbenv
rbenv install 3.2.2
rbenv local 3.2.2
gem install bundler
bundle install

# asdf
asdf install ruby 3.2.2
asdf local ruby 3.2.2
gem install bundler
bundle install

# RVM
rvm install 3.2.2
rvm use 3.2.2
gem install bundler
bundle install
```

### Verify

```bash
ruby --version   # should match Gemfile / .ruby-version
bin/rails runner "puts Rails.version"
```

### Prevent
Commit `.ruby-version` alongside `Gemfile`. Use a CI matrix that tests the declared Ruby version only.

---

## 2. Bundler / Gem Errors

**Symptoms:**
```
Could not find gem 'pg (~> 1.5)' in locally installed gems.
Bundler::GemNotFound
LoadError: cannot load such file -- some_gem
```

### Decision tree

```
Bundler / gem error
  │
  ├─ Run: bundle install
  │     If it succeeds → restart the app, done.
  │
  ├─ bundle install fails: "Could not find compatible versions"
  │     Fix: bundle update <conflicting-gem>
  │          Or pin the gem to a compatible version in Gemfile.
  │
  ├─ bundle install fails: native extension build error
  │     Examples: pg, nokogiri, sassc
  │     Fix (pg):      brew install postgresql  (macOS)
  │                    apt install libpq-dev     (Ubuntu/Debian)
  │     Fix (nokogiri): gem install nokogiri -- --use-system-libraries
  │     Fix (sassc):    brew install gcc  or  xcode-select --install
  │
  ├─ Correct Ruby but wrong gemset?
  │     Check: gem env  (look at GEM PATHS)
  │     Fix:  gem install bundler && bundle install
  │
  └─ Gemfile.lock out of sync with Gemfile?
        Fix: bundle install (updates the lock file)
             If lock file is read-only in CI, commit the updated lock.
```

### Verify

```bash
bundle check       # all gems satisfied?
bin/rails runner "puts 'gems OK'"
```

### Prevent
Commit `Gemfile.lock`. Run `bundle check` as a CI gate. Pin major versions with `~>` for stability.

---

## 3. Database Connection on Boot

**Symptoms:**
```
ActiveRecord::ConnectionNotEstablished
PG::ConnectionBad: could not connect to server
Can't connect to local MySQL server through socket
ActiveRecord::NoDatabaseError
```

### Decision tree

```
DB connection error on boot
  │
  ├─ Is the database server running?
  │     macOS (PostgreSQL): brew services start postgresql@16
  │     macOS (MySQL):      brew services start mysql
  │     Linux:              sudo systemctl start postgresql
  │     Docker:             docker compose up -d db
  │
  ├─ Does the database exist?
  │     Check: bin/rails db:migrate:status (will also fail if DB absent)
  │     Fix:   bin/rails db:create && bin/rails db:migrate
  │
  ├─ Are credentials correct?
  │     Check: config/database.yml — host, port, database, username, password
  │     Check: DATABASE_URL environment variable (overrides database.yml)
  │     Fix:   correct the values; restart the app.
  │
  ├─ Using database.yml with ERB interpolation (<%= ENV["DB_PASSWORD"] %>)?
  │     Check: printenv DB_PASSWORD (must be set in the shell)
  │     Fix:   export DB_PASSWORD=... or add to .env file (via dotenv gem).
  │
  └─ Connection refused on a non-default port?
        Check: pg_isready -h localhost -p 5432
        Fix:   update port in database.yml or DATABASE_URL.
```

### Verify

```bash
bin/rails db:migrate:status   # succeeds when DB is reachable and up to date
bin/rails runner "puts ActiveRecord::Base.connection.current_database"
```

### Prevent
Use `bin/rails db:prepare` in CI — it is idempotent (creates if missing, migrates). Document local database setup in the project README.

---

## 4. Missing Environment Variables

**Symptoms:**
```
KeyError: key not found: "STRIPE_SECRET_KEY"
RuntimeError: ENV['SECRET'] must be set
ArgumentError: Missing required config
```

### Decision tree

```
Missing ENV var
  │
  ├─ Which variable is missing?
  │     Read the error message and backtrace — it names the variable.
  │
  ├─ Where should the variable come from?
  │     Development: .env file loaded by dotenv gem (require 'dotenv/load')
  │     Test:        .env.test or test setup
  │     Production:  hosting platform env vars (Heroku, Render, Fly.io, etc.)
  │
  ├─ Using dotenv?
  │     Check: gem 'dotenv-rails' is in Gemfile (development, test group)
  │     Check: .env file exists in project root
  │     Fix:   add the variable to .env (never commit .env to git)
  │
  ├─ Using Rails credentials?
  │     Fix:   EDITOR=nano bin/rails credentials:edit
  │            Access via Rails.application.credentials.stripe[:secret_key]
  │
  └─ Initializer calls ENV.fetch(:FOO) too eagerly?
        Fix:   wrap the fetch in a conditional or lazy initializer:
               config.after_initialize { setup_stripe }
```

### Example .env file

```bash
# .env — never commit this file
DATABASE_URL=postgresql://localhost/myapp_development
REDIS_URL=redis://localhost:6379/0
STRIPE_SECRET_KEY=sk_test_...
AWS_ACCESS_KEY_ID=AKIA...
```

### Verify

```bash
printenv | grep STRIPE     # is the variable in the environment?
bin/rails runner "puts ENV['STRIPE_SECRET_KEY'].present?"
```

### Prevent
Maintain a `.env.example` file (committed, with placeholder values) so new developers know which variables are required. Use a tool like `dotenv-linter` to validate `.env` format.

---

## 5. Initializer Errors

**Symptoms:**
```
RuntimeError (raised in config/initializers/redis.rb:5)
NoMethodError (config/initializers/stripe.rb:3)
NameError: uninitialized constant MyService (config/initializers/setup.rb:2)
```

### Decision tree

```
Exception in initializer
  │
  ├─ Read the backtrace — which initializer file and line?
  │
  ├─ Is the error a NameError / uninitialized constant?
  │     Cause: autoloading is not fully set up when initializers run.
  │     Fix:   wrap the code in a to_prepare block (runs after autoloading):
  │              Rails.application.config.to_prepare do
  │                MyService.configure(...)
  │              end
  │
  ├─ Is the error a missing ENV variable (covered in section 4)?
  │     Fix:   see section 4.
  │
  ├─ Is the error a gem API change?
  │     Check: gem changelog for breaking changes in the new version.
  │     Fix:   update the initializer to use the new API, or pin the gem.
  │
  └─ Is the error a network/service call in the initializer?
        Cause: initializers should not make network calls — the service may
               be unavailable at boot time.
        Fix:   move the call to a lazy accessor or health-check endpoint.
```

### Verify

```bash
bin/rails runner "puts 'initializers OK'"
```

### Prevent
Keep initializers simple — they should register configuration, not perform I/O. Test initializer behaviour by booting in CI.

---

## 6. Asset Compilation Errors

**Symptoms:**
```
ExecJS::RuntimeError
Sprockets::FileNotFound: couldn't find file 'application'
Sass::SyntaxError
Error: Cannot find module './missing-module'
```

### Decision tree

```
Asset compilation error
  │
  ├─ ExecJS::RuntimeError
  │     Cause: Node.js is not installed or the wrong version.
  │     Check: node --version (must be >= 14 for most setups)
  │     Fix:   install or upgrade Node.js (nvm install 20)
  │
  ├─ Sprockets::FileNotFound
  │     Check: does the file exist at app/assets/ or vendor/assets/?
  │     Check: is it listed in config/initializers/assets.rb (Rails.application.config.assets.precompile)?
  │     Fix:   create the missing file or add it to the precompile list.
  │
  ├─ Sass/SCSS SyntaxError
  │     Read the file and line number in the error.
  │     Fix:   correct the CSS/SCSS syntax.
  │
  ├─ Using esbuild / Webpack / Vite (via jsbundling-rails)?
  │     Check: does node_modules/ exist?  (yarn install / npm install)
  │     Fix:   yarn install or npm install, then retry.
  │
  └─ Error: Cannot find module
        Cause: a JS import cannot be resolved.
        Fix:   yarn add <package> or correct the import path.
```

### Verify

```bash
# Attempt a full precompile
RAILS_ENV=production bin/rails assets:precompile

# Or in development just load the page — Sprockets will compile on demand
```

### Prevent
Run `bin/rails assets:precompile` in CI as a build step. Pin Node.js version in `.nvmrc` or `.node-version`.

---

## 7. Spring Issues (Rails 7)

Spring is a process preloader that speeds up development boot times. It can serve stale code if its process is not restarted after significant changes.

**Symptoms:**
- Code changes are not reflected after save
- Tests pass locally but not in CI (Spring not used in CI)
- `bin/rails` commands behave unexpectedly after a gem update or config change
- `undefined method` errors that disappear after a full server restart

### Decision tree

```
Suspected Spring issue
  │
  ├─ Stop Spring and retry
  │     bin/spring stop
  │     bin/rails server  (Spring will restart automatically)
  │
  ├─ Is Spring installed?
  │     Check: bundle exec spring status
  │     Fix:   gem 'spring' in Gemfile (development group only)
  │
  ├─ After a gem update or Gemfile change?
  │     Fix:   bin/spring stop  — Spring will pick up the new bundle on next boot.
  │
  ├─ After changing config/application.rb or an initializer?
  │     Fix:   bin/spring stop — config is not hot-reloaded by Spring.
  │
  └─ Spring keeps crashing or causing weird behaviour?
        Fix:   remove Spring from Gemfile (it is optional in Rails 7).
               Rails 7 is fast enough to boot without it on modern hardware.
```

### Verify

```bash
bin/spring stop
bin/rails runner "puts Rails.version"   # boots cleanly from scratch
```

### Remove Spring entirely (optional)

```ruby
# Gemfile — remove these lines
gem "spring"
gem "spring-watcher-listen"
```

```bash
bundle install
# Remove spring binstubs if present
grep -l spring bin/* | xargs sed -i '' '/spring/d'
```

---

## Boot Failure Checklist

Use this checklist for a systematic first pass before diving into a specific branch:

```
[ ] ruby --version matches .ruby-version and Gemfile
[ ] bundle check (or bundle install if not passing)
[ ] bin/rails db:migrate:status (no pending migrations, DB reachable)
[ ] All required ENV vars are set (check .env.example)
[ ] bin/rails runner "puts 'boot OK'" succeeds
[ ] No Spring process serving stale code (bin/spring stop)
[ ] node --version >= required (if using asset pipeline with Node)
```
