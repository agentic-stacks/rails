# Structured Logging

Rails' default log format is human-readable but noisy — one request produces dozens of lines. Structured logging collapses each request to one parseable line, which log aggregators (Datadog, Splunk, Loki, CloudWatch) can index and query.

## Lograge

Lograge replaces Rails' multi-line request logging with a single line per request.

**Install:**

```ruby
# Gemfile
gem "lograge"
```

**Enable:**

```ruby
# config/environments/production.rb
config.lograge.enabled = true
```

Enable in development too when you want cleaner output:

```ruby
# config/environments/development.rb
config.lograge.enabled = true
```

Default output (key=value format):

```
method=GET path=/orders format=html controller=OrdersController action=index status=200 duration=42.33 view=38.12 db=3.21
```

## Lograge Custom Options

Add fields to every request log line with `custom_options`. The block receives the `ActionDispatch::Request`-based event:

```ruby
# config/environments/production.rb
config.lograge.enabled = true

config.lograge.custom_options = lambda do |event|
  {
    request_id: event.payload[:headers]["X-Request-Id"],
    user_id:    event.payload[:user_id],
    ip:         event.payload[:remote_ip],
    params:     event.payload[:params].except("controller", "action", "format")
  }
end
```

Pass additional data from the controller into the payload using `append_info_to_payload`:

```ruby
# app/controllers/application_controller.rb
def append_info_to_payload(payload)
  super
  payload[:user_id]    = current_user&.id
  payload[:remote_ip]  = request.remote_ip
  payload[:request_id] = request.request_id
end
```

## JSON Output

Switch to JSON format so log aggregators parse fields automatically:

```ruby
# config/environments/production.rb
config.lograge.enabled    = true
config.lograge.formatter  = Lograge::Formatters::Json.new
```

Output:

```json
{"method":"GET","path":"/orders","format":"html","controller":"OrdersController","action":"index","status":200,"duration":42.33,"view":38.12,"db":3.21,"request_id":"abc-123","user_id":7}
```

Other built-in formatters: `Lograge::Formatters::KeyValue` (default), `Lograge::Formatters::Logstash`, `Lograge::Formatters::Graylog2`.

## Tagged Logging

`ActiveSupport::TaggedLogging` wraps any logger and prepends bracketed tags to every message. Use it to thread a request ID, user ID, or tenant through all log lines in a request.

```ruby
# config/application.rb
config.log_tags = [:request_id]
```

This tags every log line with the request ID automatically:

```
[abc-def-123] Processing by OrdersController#create as HTML
[abc-def-123] User Load (1.2ms)  SELECT * FROM users WHERE id = 7
```

Tag with arbitrary lambdas:

```ruby
config.log_tags = [
  :request_id,
  ->(req) { req.env["rack.session"]&.dig(:user_id).to_s }
]
```

Tag manually inside application code:

```ruby
Rails.logger.tagged("Payment", "stripe") do
  Rails.logger.info "Charge created: #{charge.id}"
end
# => [Payment] [stripe] Charge created: ch_abc123
```

Nest multiple tags in a single block:

```ruby
Rails.logger.tagged(current_user.id, request.request_id) do
  Rails.logger.info "Starting import"
  CsvImporter.new(file).call
  Rails.logger.info "Import complete"
end
```

## Request ID Tracing

Rails generates a unique `X-Request-Id` header for every request. Attach it to logs so you can pull all lines for a single request from your log aggregator.

Expose the request ID as a log tag (as shown above) and return it in the response:

```ruby
# config/environments/production.rb
config.log_tags = [:request_id]
```

```ruby
# config/initializers/request_id.rb
# Make ActionDispatch generate a UUID if the client does not send one
Rails.application.config.middleware.use ActionDispatch::RequestId
```

Forward the request ID into background jobs:

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  around_perform do |job, block|
    request_id = job.arguments.last.is_a?(Hash) ? job.arguments.last[:request_id] : nil
    if request_id
      Rails.logger.tagged(request_id) { block.call }
    else
      block.call
    end
  end
end
```

## Log Levels Per Environment

```ruby
# config/environments/development.rb
config.log_level = :debug     # see everything including SQL

# config/environments/test.rb
config.log_level = :warn      # suppress SQL and request noise

# config/environments/production.rb
config.log_level = :info      # debug is too verbose for production volume
```

Raise the level temporarily at runtime (no restart required):

```ruby
Rails.logger.level = Logger::WARN
```

## Silencing Noisy Logs

Silence specific actions (e.g. health check endpoints) using `lograge.ignore_actions`:

```ruby
# config/environments/production.rb
config.lograge.ignore_actions = ["HealthController#show"]
```

Silence arbitrary log output programmatically:

```ruby
Rails.logger.silence do
  # nothing logged here at :debug or :info level
  noisy_third_party_call
end
```

Suppress asset pipeline logs in development:

```ruby
# config/environments/development.rb
config.assets.quiet = true
```

## Custom JSON Logger Without Lograge

Build a lightweight JSON logger using only stdlib:

```ruby
# config/initializers/json_logger.rb
if Rails.env.production?
  logger = ActiveSupport::Logger.new($stdout)
  logger.formatter = proc do |severity, time, _progname, message|
    { level: severity, time: time.utc.iso8601(3), msg: message }.to_json + "\n"
  end
  Rails.logger = ActiveSupport::TaggedLogging.new(logger)
end
```
