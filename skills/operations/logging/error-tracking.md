# Error Tracking

Logs tell you what happened. Error tracking tells you when something breaks, groups duplicate occurrences, links to stack traces, and alerts the right person. Set up error tracking before you go live — the first production exception should never be discovered by a user.

## Why Error Tracking

- **Aggregation** — 500 occurrences of the same exception appear as one issue, not 500 log lines.
- **Context** — Captures request parameters, session data, user identity, and environment at the moment of failure.
- **Alerting** — Notifies via Slack, email, or PagerDuty when a new error type appears or a spike occurs.
- **Trends** — Shows whether error rates are rising or falling after a deploy.

## Sentry

Sentry is the most widely used error tracking service. The `sentry-rails` gem integrates automatically with Rails middleware, Active Job, and Action Mailer.

**Install:**

```ruby
# Gemfile
gem "sentry-ruby"
gem "sentry-rails"
```

**Configure:**

```ruby
# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV["SENTRY_DSN"]

  # Send errors in these environments
  config.enabled_environments = %w[staging production]

  # Attach request params and headers
  config.send_default_pii = false   # set true only if you need user data and have legal clearance

  # Capture performance traces (set 0.0–1.0; 0.1 = 10% of transactions)
  config.traces_sample_rate = 0.1

  # Filter out known noisy exceptions
  config.excluded_exceptions += ["ActionController::RoutingError", "ActiveRecord::RecordNotFound"]
end
```

`sentry-rails` hooks into the middleware stack automatically — no additional middleware configuration required.

**Add user context** so every error shows which user triggered it:

```ruby
# app/controllers/application_controller.rb
before_action :set_sentry_user_context

private

def set_sentry_user_context
  return unless current_user

  Sentry.set_user(
    id:    current_user.id,
    email: current_user.email
  )
end
```

**Add custom context** (tags and extra data):

```ruby
Sentry.set_tags(tenant: current_account.slug, plan: current_account.plan)
Sentry.set_extras(order_id: order.id, line_items: order.line_items.count)
```

**Capture exceptions manually:**

```ruby
begin
  ExternalApi.call
rescue ExternalApi::RateLimitError => e
  Sentry.capture_exception(e)
  render json: { error: "Service temporarily unavailable" }, status: :service_unavailable
end
```

**Capture a message without an exception:**

```ruby
Sentry.capture_message("Deprecated endpoint called", level: :warning, tags: { path: request.path })
```

## Honeybadger

Honeybadger is a simpler, Rails-focused alternative with a single gem.

**Install:**

```ruby
# Gemfile
gem "honeybadger"
```

**Configure:**

```bash
HONEYBADGER_API_KEY=your_key_here
```

Or generate a `config/honeybadger.yml`:

```bash
bundle exec honeybadger install YOUR_API_KEY
```

```yaml
# config/honeybadger.yml
api_key: <%= ENV["HONEYBADGER_API_KEY"] %>
environment:
  name: <%= Rails.env %>
report_data: <%= Rails.env.production? || Rails.env.staging? %>
```

**Add context:**

```ruby
Honeybadger.context(user_id: current_user.id, account: current_account.slug)
```

**Notify manually:**

```ruby
Honeybadger.notify(error, context: { order_id: order.id })
```

Honeybadger integrates with Rails exception handling, Active Job, and Sidekiq automatically when the gem is loaded.

## exception_notification

`exception_notification` is a lightweight option that emails (or posts to Slack/webhook) on every exception. No external service required — good for internal tools and small apps.

**Install:**

```ruby
# Gemfile
gem "exception_notification"
```

**Configure email delivery:**

```ruby
# config/initializers/exception_notification.rb
Rails.application.config.middleware.use(
  ExceptionNotification::Rack,
  email: {
    email_prefix:         "[ERROR] ",
    sender_address:       %("Error Notifier" <errors@example.com>),
    exception_recipients: %w[team@example.com]
  }
)
```

**Configure Slack:**

```ruby
Rails.application.config.middleware.use(
  ExceptionNotification::Rack,
  slack: {
    webhook_url:  ENV["SLACK_WEBHOOK_URL"],
    channel:      "#errors",
    username:     "Exception Bot",
    additional_parameters: {
      mrkdwn: true
    }
  }
)
```

**Ignore specific exceptions:**

```ruby
ExceptionNotifier.ignored_exceptions += [
  "ActiveRecord::RecordNotFound",
  "ActionController::RoutingError",
  "ActionController::UnknownFormat"
]
```

**Notify from a background job:**

```ruby
rescue => e
  ExceptionNotifier.notify_exception(e, env: {}, data: { job: self.class.name })
  raise
end
```

## Error Grouping and Alerting

Most error trackers group errors by exception class and backtrace fingerprint. Customise grouping when you want distinct issues to merge or split.

In Sentry, override the fingerprint:

```ruby
Sentry.configure_scope do |scope|
  scope.set_fingerprint(["payment-gateway", exception.message])
end
```

Alert policies to configure in your error tracker:

- **New issue** — notify immediately on first occurrence of a new exception type.
- **Regression** — notify when a previously resolved issue reappears after a deploy.
- **Spike** — notify when error rate exceeds N occurrences per minute.
- **First seen in production** — catch errors that only surfaced after staging passed.

## Development Tools

In development you want a rich error page, not a tracking service.

**better_errors** replaces Rails' default error page with a full REPL and stack trace:

```ruby
# Gemfile
group :development do
  gem "better_errors"
  gem "binding_of_caller"   # enables the live REPL inside better_errors
end
```

No configuration required. `binding_of_caller` is optional but strongly recommended — it lets you evaluate Ruby in the context of any stack frame at the time of the error.

**Restrict better_errors to localhost** (it is restricted by default; verify if you run a dev server accessible from other machines):

```ruby
# config/initializers/better_errors.rb
BetterErrors::Middleware.allow_ip! "127.0.0.1" if defined?(BetterErrors)
```

Never include `better_errors` or `binding_of_caller` outside the `:development` group. The live REPL executes arbitrary Ruby on the server — exposing it in production is a critical security vulnerability.
