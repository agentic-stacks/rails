# Request Errors

Use this file when your Rails application is running but returning an HTTP error to the client. Find the status code in the table below, then follow the decision tree.

## Status Code Index

| Status | Meaning | Jump to |
|---|---|---|
| 400 | Bad Request | [400 Bad Request](#400-bad-request) |
| 401 | Unauthorized | [401 Unauthorized](#401-unauthorized) |
| 403 | Forbidden | [403 Forbidden](#403-forbidden) |
| 404 | Not Found | [404-not-found](#404-not-found) |
| 422 | Unprocessable Entity | [422 Unprocessable Entity](#422-unprocessable-entity) |
| 500 | Internal Server Error | [500 Internal Server Error](#500-internal-server-error) |
| 502 | Bad Gateway | [502-503-504](#502503504-gateway-errors) |
| 503 | Service Unavailable | [502-503-504](#502503504-gateway-errors) |
| 504 | Gateway Timeout | [502-503-504](#502503504-gateway-errors) |

---

## General Debugging Approach

Before diving into a specific status code, collect information:

```bash
# 1. Tail the log during the failing request
tail -f log/development.log

# 2. In development, the error page shows the full backtrace
#    Look for the first line that is YOUR code (not Rails/gem internals)

# 3. In production, check your error tracker (Sentry, Rollbar, Honeybadger)
#    or the server log:
tail -f log/production.log | grep -E "ERROR|FATAL"
```

### Debugging Tools

**`raise params.inspect`** — print params and halt the request immediately. Use when you need to see exactly what the controller received.

```ruby
def create
  raise params.inspect   # remove before committing
  @article = Article.new(article_params)
end
```

**`binding.irb`** — drop into an interactive console at any point in the request (development only). The request pauses and you get an IRB session in the terminal running Puma.

```ruby
def create
  binding.irb   # open a REPL here
  @article = Article.new(article_params)
end
```

**`debug` gem (Rails 7.1+)** — full debugger with breakpoints. Add `debugger` in code; run the server with `bin/rdbg -n --open -- bin/rails server` or use VS Code's Rails debugger extension.

```ruby
def create
  debugger   # pauses here when using rdbg
  @article = Article.new(article_params)
end
```

---

## 400 Bad Request

**Meaning:** The server cannot process the request because of a client error — malformed syntax, invalid parameters, or a missing required parameter.

**Common Rails cause:** `ActionController::ParameterMissing` — `params.require(:key)` was called but the key was absent.

### Decision tree

```
400 Bad Request
  │
  ├─ ActionController::ParameterMissing in logs?
  │     YES → The form or API client is not sending the expected root key.
  │           Debug: add raise params.inspect before the require call.
  │           Fix: nest the params correctly in the form or request body.
  │           See common-errors.md #3 for full tree.
  │
  ├─ ActionController::BadRequest?
  │     YES → A middleware (rack-attack, custom middleware) or explicit
  │           raise ActionController::BadRequest rejected the request.
  │           Check the middleware stack and any request validation code.
  │
  └─ Custom validation in before_action returning 400?
        YES → Inspect the before_action that calls head :bad_request.
              Add logging to see which condition triggered it.
```

### Custom error pages

```ruby
# config/application.rb
config.exceptions_app = self.routes

# config/routes.rb
match "/400", to: "errors#bad_request", via: :all

# app/controllers/errors_controller.rb
class ErrorsController < ApplicationController
  def bad_request
    render status: :bad_request
  end
end
```

---

## 401 Unauthorized

**Meaning:** Authentication is required but was not provided or was invalid. The client should authenticate and retry.

**Common Rails cause:** An authentication library (Devise, Rodauth, JWT middleware) rejected the request.

### Decision tree

```
401 Unauthorized
  │
  ├─ Using Devise?
  │     Check: is authenticate_user! before_action present on this controller?
  │     Check: is the user logged in / session valid?
  │     Fix:   ensure the session cookie is being sent (check browser DevTools).
  │
  ├─ API with token / JWT authentication?
  │     Check: is the Authorization header present and correctly formatted?
  │            Authorization: Bearer <token>
  │     Check: is the token expired?
  │     Fix:   refresh the token or issue a new one.
  │     Debug: add logging in the authentication middleware to print the header.
  │
  ├─ HTTP Basic authentication?
  │     Check: is the correct username/password being sent?
  │     Fix:   correct the credentials or update the expected values.
  │
  └─ Warden / Rodauth rejecting the session?
        Check: warden.authenticated?(:user) in a controller action.
        Add logging: warden.on_failure { |env| Rails.logger.warn env.inspect }
```

### Debug tip

```ruby
# In ApplicationController — log auth state for every request
before_action do
  Rails.logger.debug "Auth: #{current_user&.id || 'anonymous'}"
end
```

---

## 403 Forbidden

**Meaning:** The server understood the request and the user is authenticated, but they do not have permission to access the resource.

**Common Rails cause:** An authorisation library (Pundit, CanCanCan, custom `authorize!`) rejected the request.

### Decision tree

```
403 Forbidden
  │
  ├─ Using Pundit?
  │     Exception: Pundit::NotAuthorizedError
  │     Check: which policy and rule? The exception message names them.
  │            e.g. "not allowed to update? this Article"
  │     Debug: policy = ArticlePolicy.new(current_user, @article)
  │            policy.update?  # returns true/false
  │     Fix:   update the policy method to grant access for the correct roles,
  │            or ensure the user has the correct role.
  │
  ├─ Using CanCanCan?
  │     Exception: CanCan::AccessDenied
  │     Check: ability.can?(:update, @article)
  │     Fix:   update app/models/ability.rb for the correct role.
  │
  ├─ Custom authorize! before_action?
  │     Read the before_action — which condition triggers the 403?
  │     Debug: add logging before the render :forbidden call.
  │
  └─ CSRF protection rendering 403?
        Check: is this a non-GET request from a browser?
        See common-errors.md #12 (InvalidAuthenticityToken).
```

### Rescue and render a custom 403

```ruby
# app/controllers/application_controller.rb
rescue_from Pundit::NotAuthorizedError do |e|
  Rails.logger.warn "403 for #{current_user&.id}: #{e.message}"
  render file: Rails.public_path.join("403.html"), status: :forbidden, layout: false
end
```

---

## 404 Not Found

**Meaning:** The server cannot find the requested resource. The resource may never have existed, or may have been deleted.

**Common Rails causes:** `ActiveRecord::RecordNotFound`, `ActionController::RoutingError`, wrong URL.

### Decision tree

```
404 Not Found
  │
  ├─ RoutingError (no route matches [GET] "/articles/foo")?
  │     Check: bin/rails routes | grep articles
  │     Fix:   correct the URL, or add the missing route.
  │     See: common-errors.md #6
  │
  ├─ RecordNotFound (Article.find raised an exception)?
  │     Check: does the record with that id exist?
  │            Article.find_by(id: params[:id])  # nil if missing
  │     Fix:   404 is the correct behaviour — ensure Rails is rescuing
  │            RecordNotFound and rendering 404 (it does by default in production).
  │
  ├─ Static file not found?
  │     Check: does the file exist in public/?
  │     Fix:   add the file or fix the URL.
  │
  └─ Wildcard route swallowing all requests?
        Check: is there a catch-all route at the bottom of routes.rb?
               match '*path', to: 'errors#not_found', via: :all
        Fix:   ensure specific routes appear before the wildcard.
```

### Custom 404 page

```ruby
# config/routes.rb — must be the last route
match "*path", to: "errors#not_found", via: :all

# app/controllers/errors_controller.rb
def not_found
  render status: :not_found
end
```

---

## 422 Unprocessable Entity

**Meaning:** The request was well-formed but the server could not process it due to semantic errors — most commonly a form submission that failed validation.

**Common Rails causes:**
- Form re-render after validation failure (Rails 7 default with Turbo)
- `protect_from_forgery` rejecting a request (`InvalidAuthenticityToken` → 422)
- Explicit `render status: :unprocessable_entity`

### Decision tree

```
422 Unprocessable Entity
  │
  ├─ Form submission failing validation (Turbo / HTML)?
  │     Rails 7 + Turbo: the controller must render 422 on validation failure
  │     so Turbo replaces the form with error messages.
  │     Check: is the controller rendering with status: :unprocessable_entity?
  │
  │     # Correct Rails 7 pattern:
  │     def create
  │       @article = Article.new(article_params)
  │       if @article.save
  │         redirect_to @article
  │       else
  │         render :new, status: :unprocessable_entity
  │       end
  │     end
  │
  ├─ CSRF token rejected (InvalidAuthenticityToken)?
  │     See common-errors.md #12.
  │     Rails maps InvalidAuthenticityToken to 422 by default.
  │     Fix: ensure csrf_meta_tags is in layout and the token is sent.
  │
  ├─ API endpoint returning 422 for validation errors?
  │     Check: the response body — it should contain the error messages.
  │     This is correct REST behaviour for validation failures.
  │     If the client is confused, ensure it reads response.errors.
  │
  └─ Explicit head :unprocessable_entity in before_action?
        Read the before_action to understand why it is being triggered.
```

### Validation errors in API responses

```ruby
def create
  @article = Article.new(article_params)
  if @article.save
    render json: @article, status: :created
  else
    render json: { errors: @article.errors }, status: :unprocessable_entity
  end
end
```

---

## 500 Internal Server Error

**Meaning:** The server encountered an unexpected error while processing the request. In Rails, this means an unhandled exception was raised.

**This is always a bug.** The goal is to find the exception, understand its cause, and fix or handle it.

### Decision tree

```
500 Internal Server Error
  │
  ├─ Step 1: Find the exception
  │     Development: the browser error page shows the exception class, message,
  │                  and full backtrace.
  │     Production:  check your error tracker (Sentry, Rollbar, Honeybadger)
  │                  or: grep "ERROR\|FATAL" log/production.log | tail -50
  │
  ├─ Step 2: Identify the exception class
  │     → NoMethodError nil:NilClass    see common-errors.md #1
  │     → RecordNotFound               see common-errors.md #2
  │     → StatementInvalid             see common-errors.md #10
  │     → NameError uninitialized      see common-errors.md #5
  │     → Any other exception          read the backtrace (step 3)
  │
  ├─ Step 3: Read the backtrace
  │     Find the first frame that is YOUR application code (not a gem or Rails).
  │     That line is almost always where the problem originates.
  │
  ├─ Step 4: Reproduce locally
  │     bin/rails server
  │     Visit the same URL / submit the same form.
  │     Add raise params.inspect or binding.irb to inspect state.
  │
  └─ Step 5: Fix the exception
        Either fix the root cause or rescue and handle gracefully:
        rescue SomeError => e
          Rails.logger.error e.message
          render :error, status: :internal_server_error
```

### Set up an error tracker

```ruby
# Gemfile
gem "sentry-rails"
gem "sentry-ruby"

# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = Rails.application.credentials.sentry[:dsn]
  config.breadcrumbs_logger = [:active_support_logger, :http_logger]
  config.traces_sample_rate = 0.1
end
```

### Custom 500 page

```html
<!-- public/500.html — served by the web server, not Rails -->
<!DOCTYPE html>
<html>
  <head><title>We'll be right back</title></head>
  <body><h1>Something went wrong. We've been notified.</h1></body>
</html>
```

---

## 502/503/504 Gateway Errors

These errors are returned by your **web server or load balancer** (Nginx, Caddy, AWS ALB), not by Rails itself. Rails either did not respond at all, responded too slowly, or is not running.

### Status meaning

| Status | Meaning | Typical cause |
|---|---|---|
| 502 Bad Gateway | Upstream returned an invalid response | Puma crashed, sent a bad response |
| 503 Service Unavailable | Upstream is down | Puma not running, all workers busy |
| 504 Gateway Timeout | Upstream did not respond in time | Request took too long, Puma hung |

### Decision tree

```
502 / 503 / 504
  │
  ├─ Step 1: Is the Rails process running?
  │     Check: ps aux | grep puma
  │     Fix 503: start the app (bin/rails server or systemctl start myapp)
  │
  ├─ Step 2: Did Puma crash?
  │     Check: journalctl -u myapp -n 100  or  tail log/production.log
  │     Fix:   fix the crash (see 500 section) and restart.
  │
  ├─ 504 Gateway Timeout — request is too slow
  │     Check: which endpoint? Is it always slow or intermittently?
  │     Quick fix: increase Nginx proxy_read_timeout or ALB idle timeout.
  │     Root fix: profile the slow request (see performance-issues.md).
  │
  ├─ 503 all workers busy (Puma thread pool exhausted)?
  │     Check: Puma dashboard or metrics — are all threads active?
  │     Short-term: increase Puma workers/threads in config/puma.rb.
  │     Root fix: reduce request duration (see performance-issues.md).
  │
  └─ 502 bad response from Puma?
        Check: is Puma returning a non-HTTP response (binary garbage, partial body)?
        Cause: usually an unhandled exception in Rack middleware, or a signal
               that killed a worker mid-response.
        Fix:   check production.log for the exception immediately before the 502.
```

### Puma configuration

```ruby
# config/puma.rb
max_threads_count = ENV.fetch("RAILS_MAX_THREADS", 5)
min_threads_count = ENV.fetch("RAILS_MIN_THREADS") { max_threads_count }
threads min_threads_count, max_threads_count

worker_timeout 3600 if ENV.fetch("RAILS_ENV", "development") == "development"

port ENV.fetch("PORT", 3000)

environment ENV.fetch("RAILS_ENV", "development")

pidfile ENV.fetch("PIDFILE", "tmp/pids/server.pid")

# Production: use multiple workers (forked processes)
workers ENV.fetch("WEB_CONCURRENCY", 2)

preload_app!
```

### Quick health check endpoint

```ruby
# config/routes.rb
get "/up", to: "rails/health#show", as: :rails_health_check  # Rails 7.1+

# Or custom:
get "/health", to: proc { [200, {}, ["OK"]] }
```

---

## Error Monitoring Setup

For all 5xx errors in production, you need an error tracker to capture the full context (backtrace, user, params, environment). Options:

| Service | Gem | Free tier |
|---|---|---|
| Sentry | `sentry-rails` | Yes |
| Rollbar | `rollbar` | Yes |
| Honeybadger | `honeybadger` | Yes |
| Bugsnag | `bugsnag` | Yes |
| AppSignal | `appsignal` | No |

Minimum viable setup — log all unhandled exceptions with context:

```ruby
# app/controllers/application_controller.rb
rescue_from StandardError do |e|
  Rails.logger.error([
    e.class,
    e.message,
    e.backtrace.first(10).join("\n")
  ].join("\n"))
  raise  # re-raise so Rails renders the correct error page
end
```
