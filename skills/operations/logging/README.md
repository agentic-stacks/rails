# Logging

Observability in Rails rests on three pillars:

| Pillar | What it answers | Rails tooling |
|---|---|---|
| **Logs** | What happened and when | `Rails.logger`, Lograge, tagged logging |
| **Metrics** | How much and how fast | StatsD, Prometheus, ActiveSupport::Notifications |
| **Traces** | Where time was spent across services | OpenTelemetry, Skylight, Scout APM |

This skill covers logs. For metrics and traces see `performance/`.

## Rails.logger

`Rails.logger` is available everywhere in your application — models, controllers, jobs, mailers, and initializers.

```ruby
Rails.logger.debug  "Loaded #{users.count} users"
Rails.logger.info   "Order #{order.id} placed by user #{user.id}"
Rails.logger.warn   "Deprecated call to legacy_sync — use async path"
Rails.logger.error  "Stripe webhook failed: #{e.message}"
Rails.logger.fatal  "Database connection pool exhausted — restarting"
```

Inside a controller or model, `logger` is available directly (it delegates to `Rails.logger`):

```ruby
class OrdersController < ApplicationController
  def create
    logger.info "Creating order for user #{current_user.id}"
    # ...
  end
end
```

## Log Levels

Rails defines six levels in ascending severity:

| Level | Value | Use for |
|---|---|---|
| `debug` | 0 | Verbose diagnostic output (SQL, cache hits, variable dumps) |
| `info`  | 1 | Normal operations (requests, job execution, user actions) |
| `warn`  | 2 | Unexpected but recoverable conditions |
| `error` | 3 | Errors that need attention but don't stop the process |
| `fatal` | 4 | Unrecoverable errors — process will likely exit |
| `unknown` | 5 | Messages that must always appear regardless of level |

The configured level acts as a floor: messages below it are silenced. Setting `:warn` in production means `debug` and `info` calls produce no output.

## Configure Log Level

Set a single level for all environments:

```ruby
# config/application.rb
config.log_level = :info
```

Override per environment (the common pattern):

```ruby
# config/environments/development.rb
config.log_level = :debug

# config/environments/test.rb
config.log_level = :warn   # reduce noise in test output

# config/environments/production.rb
config.log_level = :info
```

Change the level at runtime without a restart:

```ruby
Rails.logger.level = Logger::DEBUG
```

## Log Output Destination

Rails logs to `log/<environment>.log` by default. Redirect to stdout for containerised deployments (stdout is captured by Docker, Heroku, and Kubernetes log drivers):

```ruby
# config/environments/production.rb
config.logger = ActiveSupport::Logger.new($stdout)
config.log_level = :info
```

Or use a tagged, multi-destination logger:

```ruby
# config/application.rb
logger = ActiveSupport::Logger.new($stdout)
logger.formatter = config.log_formatter
config.logger = ActiveSupport::TaggedLogging.new(logger)
```

## Files in This Skill

| File | What it covers |
|---|---|
| `instrumentation.md` | ActiveSupport::Notifications, custom events, LogSubscriber |
| `structured-logging.md` | Lograge, tagged logging, JSON output, request ID tracing |
| `error-tracking.md` | Sentry, Honeybadger, exception_notification, better_errors |

## Next Steps

- Add structure to request logs: `structured-logging.md`
- Emit and subscribe to custom events: `instrumentation.md`
- Track exceptions in production: `error-tracking.md`
