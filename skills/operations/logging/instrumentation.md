# Instrumentation

ActiveSupport::Notifications is Rails' internal pub/sub bus. Rails fires events for every SQL query, controller action, cache operation, and mailer delivery. You can subscribe to those events, fire your own, and build structured log subscribers on top of them.

## Subscribe to an Event

`ActiveSupport::Notifications.subscribe` takes a pattern and a block. The block receives five arguments:

```ruby
ActiveSupport::Notifications.subscribe("sql.active_record") do |name, started, finished, id, payload|
  duration_ms = ((finished - started) * 1000).round(2)
  Rails.logger.debug "[SQL] #{payload[:sql]} (#{duration_ms}ms)"
end
```

| Argument | Type | Value |
|---|---|---|
| `name` | String | Event name, e.g. `"sql.active_record"` |
| `started` | Time | When the event began |
| `finished` | Time | When the event ended |
| `id` | String | Unique event ID |
| `payload` | Hash | Event-specific data |

Subscribe with a regex to catch a family of events:

```ruby
ActiveSupport::Notifications.subscribe(/\.action_mailer$/) do |name, started, finished, _id, payload|
  Rails.logger.info "Mailer event: #{name} to #{payload[:to]}"
end
```

## Built-in Rails Events

### SQL — `sql.active_record`

| Payload key | Value |
|---|---|
| `:sql` | The SQL string |
| `:name` | Query name (e.g. `"User Load"`) |
| `:binds` | Bound parameters |
| `:cached` | `true` when served from query cache |

### Controller actions — `process_action.action_controller`

| Payload key | Value |
|---|---|
| `:controller` | Controller class name |
| `:action` | Action name |
| `:format` | Response format (`:html`, `:json`, …) |
| `:method` | HTTP method |
| `:path` | Request path |
| `:status` | HTTP status code |
| `:view_runtime` | Time spent in views (ms) |
| `:db_runtime` | Time spent in the database (ms) |

### Render — `render_template.action_view`

| Payload key | Value |
|---|---|
| `:identifier` | Template file path |
| `:layout` | Layout file path |

### Cache — `cache_read.active_support`, `cache_write.active_support`, `cache_fetch_hit.active_support`

| Payload key | Value |
|---|---|
| `:key` | Cache key string |
| `:store` | Store class name |
| `:hit` | (fetch only) whether the key was present |

### Mailer — `deliver.action_mailer`

| Payload key | Value |
|---|---|
| `:mailer` | Mailer class name |
| `:message_id` | Message-ID header value |
| `:to` | Recipients array |
| `:subject` | Email subject |
| `:perform_deliveries` | Whether delivery was attempted |

## Instrument Custom Events

Wrap any block of code to emit a named event. The block's return value is returned by `instrument`.

```ruby
ActiveSupport::Notifications.instrument("import.my_app", rows: csv.count, source: filename) do
  CsvImporter.new(csv).call
end
```

Read the payload inside the block to add runtime data (mutation is safe):

```ruby
ActiveSupport::Notifications.instrument("payment.my_app", provider: "stripe") do |payload|
  result = StripeGateway.charge(amount: order.total)
  payload[:charge_id] = result.id
  payload[:status]    = result.status
end
```

Subscribe to your custom event the same way as built-in ones:

```ruby
ActiveSupport::Notifications.subscribe("payment.my_app") do |name, started, finished, _id, payload|
  duration_ms = ((finished - started) * 1000).round(2)
  Rails.logger.info "[Payment] provider=#{payload[:provider]} charge=#{payload[:charge_id]} status=#{payload[:status]} duration=#{duration_ms}ms"
end
```

Place subscriptions in an initializer so they register once at boot:

```ruby
# config/initializers/notifications.rb
ActiveSupport::Notifications.subscribe("payment.my_app") do |name, started, finished, _id, payload|
  # ...
end
```

## LogSubscriber

`ActiveSupport::LogSubscriber` is a structured base class for subscribers. It binds to the logger automatically and gives you helper methods (`color`, `info`, `debug`). Rails uses it internally for all its own log output.

```ruby
# app/subscribers/import_log_subscriber.rb
class ImportLogSubscriber < ActiveSupport::LogSubscriber
  def import(event)
    info do
      rows     = event.payload[:rows]
      duration = event.duration.round(2)
      "Import completed: #{rows} rows in #{duration}ms"
    end
  end
end

ImportLogSubscriber.attach_to :my_app
```

`attach_to :my_app` subscribes to all events whose name ends in `.my_app`. The method name must match the prefix: the event `import.my_app` calls the `import` method.

`event.duration` is the elapsed time in milliseconds (a float). `event.payload` is the hash passed to `instrument`.

## Unsubscribing

Store the subscriber object returned by `subscribe` if you need to remove it later:

```ruby
sub = ActiveSupport::Notifications.subscribe("sql.active_record") { |*args| }
# … later …
ActiveSupport::Notifications.unsubscribe(sub)
```

This is useful in tests where you want to listen only inside a specific block.

## Monotonic Timing

For performance-sensitive subscribers, use `monotonic_subscribe` to avoid clock skew from system time adjustments:

```ruby
ActiveSupport::Notifications.monotonic_subscribe("sql.active_record") do |name, started, finished, _id, payload|
  duration_ms = ((finished - started) * 1000).round(2)
  StatsD.timing("sql.duration", duration_ms)
end
```
