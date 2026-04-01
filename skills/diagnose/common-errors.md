# Common Rails Errors

Top 20 exception classes you will encounter in Rails applications, each with a decision tree following the symptom → check → fix → prevent pattern.

---

## 1. NoMethodError: undefined method `...' for nil:NilClass

### What it means
You called a method on a variable that is `nil`. The object you expected to be present was not found, never assigned, or was not loaded from the database.

### Decision tree

```
NoMethodError nil:NilClass
  │
  ├─ Is the variable an ActiveRecord object loaded with find_by?
  │     YES → find_by returns nil when no record matches.
  │           Fix: use find (raises RecordNotFound) or guard with &. or if/unless.
  │           Fix: add a before_action that redirects or 404s when object is nil.
  │
  ├─ Is the variable set in a controller before_action?
  │     YES → Check that the before_action runs for this action (only:/except:).
  │           Check that the instance variable name matches (@article vs @post).
  │
  ├─ Is the variable from an association (user.profile.avatar_url)?
  │     YES → The association itself may be nil (no profile record).
  │           Fix: use safe navigation: user.profile&.avatar_url
  │           Fix: build the association on user creation.
  │
  └─ Is the variable from a hash or params?
        YES → The key may be absent or misspelled.
              Fix: use params[:key] with a guard, or params.fetch(:key, default).
```

### Fix examples

```ruby
# Bad
@article.title

# Good — raises 404 automatically
@article = Article.find(params[:id])

# Good — safe navigation for optional associations
user.profile&.bio

# Good — guard clause in controller
def show
  @article = Article.find_by(id: params[:id])
  return render :not_found, status: :not_found unless @article
end
```

### Prevent
Enable `strict_loading` in development to surface missing associations early. Write tests that exercise the nil path.

---

## 2. ActiveRecord::RecordNotFound

### What it means
`find`, `find_by!`, or `find_sole_by` was called and no record matched the given id or conditions.

### Decision tree

```
RecordNotFound
  │
  ├─ Is this in a controller action using find(params[:id])?
  │     YES → Rails rescues this automatically and renders a 404 in production.
  │           In development you see the full error page — this is expected.
  │           Fix: ensure your routes are correct and the id is valid.
  │
  ├─ Is the record supposed to exist (was just created)?
  │     YES → Check for pending migrations (bin/rails db:migrate:status).
  │           Check that the seed or fixture data was loaded.
  │           Check that the environment is correct (development vs test db).
  │
  └─ Is this in a background job?
        YES → The record may have been deleted between enqueue and execution.
              Fix: rescue RecordNotFound in the job and discard or retry:
              rescue_from ActiveRecord::RecordNotFound, with: :discard
```

### Fix

```ruby
# Automatic 404 in controllers — nothing to change
@article = Article.find(params[:id])

# In jobs — discard if the record is gone
class SendEmailJob < ApplicationJob
  discard_on ActiveRecord::RecordNotFound
  def perform(user_id)
    user = User.find(user_id)
    UserMailer.welcome(user).deliver_now
  end
end
```

### Prevent
Use `find` (not `find_by`) when the absence of a record is exceptional. Add database-level foreign key constraints so dependent records cannot be deleted unexpectedly.

---

## 3. ActionController::ParameterMissing

### What it means
`params.require(:key)` was called but `:key` was absent from the request parameters. This produces a 400 Bad Request in production.

### Decision tree

```
ParameterMissing
  │
  ├─ Is the form sending the right root key?
  │     Check: in controller, add raise params.inspect before the require call.
  │     If params shows flat keys (title: "..") instead of (article: {title: ".."}):
  │       Fix: ensure the form uses fields_for / form_with model: @article,
  │            or that the JSON body uses {"article": {...}}.
  │
  ├─ Is this an API client (Postman, curl, fetch)?
  │     YES → Check the Content-Type header is application/json.
  │           Check the body is nested: {"article": {"title": "Foo"}}.
  │
  └─ Is the key name correct?
        Check: compare params.require(:article) with the actual key name in the log.
        Fix: correct the key name in the strong params method or the form.
```

### Fix

```ruby
# Debug: inspect params to see what arrived
def article_params
  raise params.inspect   # temporary — remove after debugging
  params.require(:article).permit(:title, :body)
end
```

### Prevent
Write request specs that POST with both valid and missing parameters. Use `:with` option to mark required form fields.

---

## 4. ActiveRecord::RecordInvalid

### What it means
`save!`, `create!`, or `update!` was called and the record failed validations. The bang methods raise; non-bang methods return `false`.

### Decision tree

```
RecordInvalid
  │
  ├─ What validations failed?
  │     Check: record.errors.full_messages in the console or logs.
  │
  ├─ Is the error in a controller action?
  │     Check: are you using save! where save is appropriate?
  │     Fix: use save and re-render the form on failure:
  │            if @article.save
  │              redirect_to @article
  │            else
  │              render :new, status: :unprocessable_entity
  │            end
  │
  ├─ Is the error in a service object or background job?
  │     Fix: rescue RecordInvalid and handle gracefully.
  │
  └─ Is this a uniqueness validation failing in tests?
        Fix: ensure test data is cleaned between tests (database_cleaner or
             transactions). Check for fixture conflicts.
```

### Fix

```ruby
# Controller — non-bang with re-render
def create
  @article = Article.new(article_params)
  if @article.save
    redirect_to @article, notice: "Created."
  else
    render :new, status: :unprocessable_entity
  end
end

# Service — rescue and return errors
def call
  record.save!
rescue ActiveRecord::RecordInvalid => e
  Result.failure(e.record.errors)
end
```

### Prevent
Test validations with unit tests. Prefer non-bang methods in controllers. Use bang methods in jobs and rescue explicitly.

---

## 5. NameError: uninitialized constant

### What it means
Rails could not find a class or module by the name you used. This is most often an autoloading issue or a missing require.

### Decision tree

```
NameError uninitialized constant Foo::Bar
  │
  ├─ Does the file exist?
  │     Check: app/models/foo/bar.rb or the equivalent path.
  │     Fix: create the file if missing.
  │
  ├─ Does the file exist but the constant is not found?
  │     Check: does the class/module declaration match the file path?
  │            app/models/foo/bar.rb must define module Foo; class Bar.
  │     Fix: fix the namespace nesting to match the file path.
  │
  ├─ Is the constant in a gem?
  │     Check: is the gem in Gemfile? Run bundle check.
  │     Fix: add the gem, run bundle install, restart the server.
  │
  ├─ Is this happening only in production?
  │     Check: is eager_load_paths configured correctly in config/application.rb?
  │     Fix: add the directory to config.eager_load_paths if it is non-standard.
  │
  └─ Is this in a Rails initializer?
        YES → Initializers run before autoloading is set up in some cases.
              Fix: wrap code in a to_prepare block or use require explicitly.
```

### Prevent
Follow Rails naming conventions precisely. Use `bin/rails zeitwerk:check` (Rails 6+) to validate autoloading before deploying.

```bash
bin/rails zeitwerk:check
```

---

## 6. ActionController::RoutingError: No route matches

### What it means
The HTTP method + path combination does not match any route defined in `config/routes.rb`.

### Decision tree

```
RoutingError
  │
  ├─ Print all routes
  │     bin/rails routes | grep articles
  │     or visit http://localhost:3000/rails/info/routes in development
  │
  ├─ Is the HTTP verb wrong?
  │     Check: a form doing a DELETE via a GET will fail.
  │     Fix: ensure method: :delete in link_to / button_to.
  │
  ├─ Is the path correct?
  │     Check: URL helper vs actual path. Use bin/rails routes to compare.
  │     Fix: use the named helper (_path / _url) instead of hardcoded strings.
  │
  ├─ Is the route defined but the controller/action missing?
  │     Check: bin/rails routes shows the route but the action does not exist.
  │     Fix: add the action to the controller.
  │
  └─ Is the route behind a scope or namespace?
        Check: admin_articles_path vs articles_path.
        Fix: use the correct namespaced helper.
```

### Prevent
Use route helpers everywhere instead of string paths. Add route coverage to integration tests.

---

## 7. ActiveRecord::PendingMigrationError

### What it means
There are migrations in `db/migrate/` that have not been run against the current database.

### Fix

```bash
# Run all pending migrations
bin/rails db:migrate

# Verify
bin/rails db:migrate:status
```

### Decision tree

```
PendingMigrationError
  │
  ├─ In development?
  │     Fix: bin/rails db:migrate — done.
  │
  ├─ In test environment?
  │     Fix: bin/rails db:migrate RAILS_ENV=test
  │          or bin/rails db:test:prepare
  │
  └─ In production (deploy pipeline)?
        Fix: run bin/rails db:migrate as part of the deploy step before
             restarting the app server.
        Check: is the DB_URL / DATABASE_URL environment variable correct?
```

### Prevent
Run `bin/rails db:migrate:status` in CI before tests. Include `db:migrate` in your deploy script before process restart.

---

## 8. ActionView::Template::Error

### What it means
An exception was raised inside an ERB, Haml, or Slim template. The actual error is wrapped — the root cause is the exception listed after "caused by".

### Decision tree

```
Template::Error
  │
  ├─ Read the "caused by" line — that is the real error.
  │     Most common: NoMethodError nil:NilClass (see error #1).
  │
  ├─ Is the instance variable nil in the view?
  │     Check: the controller action may not set @variable for all paths.
  │     Fix: ensure the before_action or action body always assigns it.
  │
  ├─ Is a partial raising the error?
  │     Check: the backtrace will show the partial path (views/articles/_form).
  │     Fix: add a guard in the partial: <% if article.persisted? %>.
  │
  └─ Is a helper method raising the error?
        Check: the backtrace will name the helper file.
        Fix: add nil checks or rescue inside the helper.
```

### Prevent
Assign sensible defaults in controllers (`@articles = []` instead of leaving it nil). Write system tests that render templates under realistic conditions.

---

## 9. LoadError: cannot load such file

### What it means
Ruby's `require` or `require_relative` could not find the file at the given path.

### Decision tree

```
LoadError
  │
  ├─ Is this a gem?
  │     Check: is the gem in Gemfile?
  │     Fix: add to Gemfile, run bundle install, restart.
  │
  ├─ Is this a lib/ file?
  │     Check: is the file in config.autoload_paths or config.eager_load_paths?
  │     Fix: add lib/ to eager_load_paths (Rails 7 does not add it by default):
  │            config.autoload_lib(ignore: %w[assets tasks])
  │
  └─ Is this a path typo?
        Check: verify the exact file name and directory.
        Fix: correct the path or rename the file to match.
```

### Prevent
In Rails 7+, use `config.autoload_lib` for files under `lib/`. Avoid raw `require` calls for application code.

---

## 10. ActiveRecord::StatementInvalid

### What it means
The SQL statement sent to the database was rejected. The message contains the database-level error (column does not exist, syntax error, type mismatch, etc.).

### Decision tree

```
StatementInvalid
  │
  ├─ "column does not exist" or "Unknown column"
  │     Fix: run the pending migration that adds the column.
  │          bin/rails db:migrate
  │
  ├─ "relation does not exist" or "Table ... doesn't exist"
  │     Fix: run pending migrations to create the table.
  │
  ├─ "PG::InvalidTextRepresentation" or type mismatch
  │     Fix: check that the value type matches the column type
  │          (e.g., passing a string to an integer column).
  │
  └─ Syntax error in SQL
        Check: use Article.where(...).to_sql to inspect the generated SQL.
        Fix: correct the Arel expression or raw SQL fragment.
```

### Prevent
Never interpolate user input directly into SQL strings. Use parameterised queries (`.where("status = ?", status)`). Run `bin/rails db:migrate:status` in CI.

---

## 11. Errno::EADDRINUSE: Address already in use

### What it means
The port Rails wants to bind to (default 3000) is already occupied by another process.

### Fix

```bash
# Find what is using port 3000
lsof -i :3000

# Kill the process (replace PID with the actual number)
kill -9 <PID>

# Or start Rails on a different port
bin/rails server -p 3001
```

### Decision tree

```
EADDRINUSE
  │
  ├─ Is a previous Rails server still running?
  │     Fix: kill the PID shown by lsof -i :3000.
  │
  ├─ Is the tmp/pids/server.pid stale?
  │     Fix: rm tmp/pids/server.pid then start again.
  │
  └─ Is another application using port 3000?
        Fix: bin/rails server -p 3001 (or any free port).
```

### Prevent
Add a `.foreman` or `Procfile.dev` that specifies the port explicitly. Use `bin/dev` (Rails 7+) which manages the PID lifecycle.

---

## 12. ActionController::InvalidAuthenticityToken

### What it means
Rails rejected a non-GET request because the CSRF token was missing or did not match the session token.

### Decision tree

```
InvalidAuthenticityToken
  │
  ├─ Is this a browser form?
  │     Check: does the form include <%= csrf_meta_tags %> in the layout?
  │     Check: does the form use form_with / form_tag (which auto-include the token)?
  │     Fix: add csrf_meta_tags to the layout head; use Rails form helpers.
  │
  ├─ Is this an API request from a JS client (fetch / Axios)?
  │     Fix: read the token from the meta tag and include it in the header:
  │            X-CSRF-Token: document.querySelector('meta[name="csrf-token"]').content
  │
  ├─ Is this an API-only controller?
  │     Fix: protect_from_forgery with: :null_session  (or skip it entirely)
  │          class ApiController < ActionController::API — CSRF is not included.
  │
  └─ Did the session expire or get cleared?
        Fix: ensure session store is working (cookie size, secret_key_base).
```

### Prevent
Use `protect_from_forgery with: :exception` (default) for HTML controllers and `:null_session` or skip for API controllers. Include `csrf_meta_tags` in every layout.

---

## 13. ActiveRecord::ConnectionNotEstablished

### What it means
Rails could not open a connection to the database. The database server may be down, the credentials wrong, or the database not yet created.

### Decision tree

```
ConnectionNotEstablished
  │
  ├─ Is the database server running?
  │     Check: pg_isready (PostgreSQL) or mysqladmin ping (MySQL).
  │     Fix: start the database server.
  │
  ├─ Is config/database.yml correct?
  │     Check: host, port, database name, username, password.
  │     Fix: correct the values and restart.
  │
  ├─ Does the database exist?
  │     Fix: bin/rails db:create then bin/rails db:migrate.
  │
  ├─ Is DATABASE_URL set in production?
  │     Check: echo $DATABASE_URL
  │     Fix: set the environment variable in your hosting environment.
  │
  └─ Is the connection pool exhausted?
        Check: your APM or DB server active connection count.
        Fix: increase pool size in database.yml or reduce connection holding time.
```

### Prevent
Use `bin/rails db:prepare` in CI and deploy scripts. Monitor connection pool utilisation. Set `checkout_timeout` in database.yml.

---

## 14. Sprockets::Rails::Helper::AssetNotFound / MissingAssetError

### What it means
A stylesheet, JavaScript, image, or font referenced in a view or asset manifest could not be found by the asset pipeline.

### Decision tree

```
MissingAssetError (asset not found: "application.css")
  │
  ├─ In development?
  │     Check: does app/assets/stylesheets/application.css exist?
  │     Check: bin/rails assets:precompile — does it raise or succeed?
  │     Fix: create the missing file or correct the require/import path.
  │
  ├─ In production?
  │     Check: were assets precompiled during the deploy? (public/assets/ populated?)
  │     Fix: run RAILS_ENV=production bin/rails assets:precompile as a deploy step.
  │
  ├─ Using Propshaft (Rails 8 default)?
  │     Check: is the file under app/assets/ or a mounted engine's assets?
  │     Fix: ensure the file exists at the expected path.
  │
  └─ Using importmaps or esbuild?
        Check: is the JS file listed in config/importmap.rb or entrypoints?
        Fix: pin the package or add it to the build entrypoint.
```

### Prevent
Run `bin/rails assets:precompile` in CI to catch missing assets before deploy. Add asset paths to integration tests.

---

## 15. ArgumentError: wrong number of arguments

### What it means
A method was called with more or fewer arguments than its signature accepts.

### Decision tree

```
ArgumentError wrong number of arguments (given N, expected M)
  │
  ├─ Read the backtrace — which method and file?
  │
  ├─ Did a gem update change a method signature?
  │     Check: git log Gemfile.lock to see what changed.
  │     Fix: update the call site to match the new signature.
  │          Or pin the gem version to the previous working version.
  │
  ├─ Is it a Rails helper call (e.g., link_to, form_with)?
  │     Check: Rails 7 changed several helper signatures.
  │     Fix: consult the Rails upgrade guide for the version you moved to.
  │
  └─ Is it a custom method?
        Fix: align the call with the method definition.
        Consider using keyword arguments with defaults to make calls flexible.
```

### Prevent
Write unit tests for changed method signatures. Freeze gem versions with exact (~>) constraints for critical dependencies.

---

## 16. FrozenError: can't modify frozen String / Object

### What it means
You attempted to mutate an object that has been frozen. In Ruby 3+, string literals can be frozen by default (if `# frozen_string_literal: true` is present). Rails freezes some internal objects.

### Decision tree

```
FrozenError
  │
  ├─ Is there # frozen_string_literal: true at the top of the file?
  │     YES → String literals in that file are frozen.
  │           Fix: use .dup to get a mutable copy, or use String.new("..."),
  │                or avoid in-place mutation (use + or interpolation instead).
  │
  ├─ Is the object an ActiveRecord attribute value?
  │     YES → ActiveRecord attribute values are frozen.
  │           Fix: call .dup before mutation:
  │                record.name.dup << " suffix"
  │
  └─ Are you mutating a constant or class-level string?
        Fix: freeze intentionally and use .dup at call sites, or use a method
             that returns a new string.
```

### Prevent
Enable `# frozen_string_literal: true` globally (rubocop can enforce this). Write code that avoids in-place string mutation.

---

## 17. ActiveRecord::NoDatabaseError

### What it means
The database does not exist yet. This is common in fresh environments or after `db:drop`.

### Fix

```bash
bin/rails db:create
bin/rails db:migrate
# or in one step (create + migrate + seed):
bin/rails db:setup
```

### Prevent
Use `bin/rails db:prepare` in CI — it creates the database if missing and runs pending migrations without failing on an already-existing database.

---

## 18. Bundler::GemNotFound / Could not find gem

### What it means
A gem listed in `Gemfile.lock` is not installed in the current Ruby environment.

### Decision tree

```
GemNotFound / Could not find gem 'foo'
  │
  ├─ Run bundle install
  │     Fix: bundle install
  │
  ├─ Using RVM / rbenv / asdf — is the right Ruby active?
  │     Check: ruby --version matches .ruby-version file.
  │     Fix: rbenv local <version>  or  asdf install ruby <version>
  │
  ├─ Is the BUNDLE_PATH set to a non-existent directory?
  │     Check: bundle env | grep BUNDLE_PATH
  │     Fix: bundle install or unset BUNDLE_PATH.
  │
  └─ Is this in a Docker or CI container?
        Fix: run bundle install as part of the image build or CI setup step.
             Cache the bundle directory between runs (vendor/bundle or BUNDLE_PATH).
```

### Prevent
Commit `Gemfile.lock`. Add `bundle check || bundle install` to your CI setup step.

---

## 19. SyntaxError

### What it means
Ruby encountered invalid syntax in a file. The app will not load until the syntax error is fixed.

### Decision tree

```
SyntaxError unexpected ')', expecting end-of-input
  │
  ├─ Read the file path and line number in the error message.
  │
  ├─ Common causes:
  │     - Missing or extra end keyword
  │     - Missing closing bracket/paren
  │     - Heredoc not terminated
  │     - Ruby 3 syntax change (e.g., no space before parenthesis)
  │
  ├─ Quick check:
  │     ruby -c app/models/article.rb
  │     # Outputs "Syntax OK" or the error location
  │
  └─ Editor not highlighting the error?
        Fix: run rubocop --only Style/Syntax on the file.
```

### Fix

```bash
# Check a single file for syntax errors
ruby -c path/to/file.rb

# Check the whole app
find app lib -name "*.rb" -exec ruby -c {} \; 2>&1 | grep -v "Syntax OK"
```

### Prevent
Use an editor with Ruby syntax highlighting and real-time error checking (Solargraph, RuboCop LSP). Add a syntax check step to CI.

---

## 20. ActiveSupport::MessageEncryptor::InvalidMessage (credentials)

### What it means
Rails could not decrypt `config/credentials.yml.enc` because the `master.key` (or the key in `RAILS_MASTER_KEY`) does not match the key used to encrypt the file.

### Decision tree

```
InvalidMessage (credentials)
  │
  ├─ Does config/master.key exist?
  │     NO  → The key was not committed (correct — it should not be).
  │            Fix: obtain master.key from a secure vault (1Password, AWS Secrets Manager)
  │                 or from a teammate, and place it in config/master.key.
  │                 Or set RAILS_MASTER_KEY environment variable.
  │
  ├─ Is RAILS_MASTER_KEY set but wrong?
  │     Check: echo $RAILS_MASTER_KEY (first 4 chars should match start of master.key).
  │     Fix: update the environment variable to the correct value.
  │
  ├─ Was credentials.yml.enc re-generated without updating the key?
  │     YES → The file was encrypted with a new key but the old key is in place.
  │            Fix: re-encrypt with the correct key, or restore credentials.yml.enc
  │                 from version control.
  │
  └─ Using per-environment credentials (credentials/production.yml.enc)?
        Fix: ensure RAILS_MASTER_KEY for that environment is set and matches
             config/credentials/production.key.
```

### Prevent
Store `master.key` in a secrets manager (never in git). Rotate credentials deliberately via `bin/rails credentials:edit`. Document the key handover process for new team members.

```bash
# Edit credentials (uses EDITOR or $VISUAL)
EDITOR=nano bin/rails credentials:edit

# Per-environment credentials (Rails 6+)
EDITOR=nano bin/rails credentials:edit --environment production
```
