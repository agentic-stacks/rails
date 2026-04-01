# Background Jobs

Active Job is Rails' queuing interface. Write a job once; swap the backend (Solid Queue, Sidekiq, GoodJob) without touching job code.

## Generate a Job

```bash
bin/rails generate job ProcessOrder
# Creates: app/jobs/process_order_job.rb
#          test/jobs/process_order_job_test.rb
```

Basic job anatomy:

```ruby
# app/jobs/process_order_job.rb
class ProcessOrderJob < ApplicationJob
  queue_as :default

  def perform(order_id)
    order = Order.find(order_id)
    order.process!
  end
end
```

Always pass IDs, not ActiveRecord objects. Objects cannot be serialised reliably across process restarts.

## Enqueue a Job

```ruby
# Enqueue immediately (runs as soon as a worker picks it up)
ProcessOrderJob.perform_later(order.id)

# Enqueue with a delay
ProcessOrderJob.set(wait: 5.minutes).perform_later(order.id)

# Enqueue at a specific time
ProcessOrderJob.set(wait_until: Date.tomorrow.noon).perform_later(order.id)

# Override the queue inline
ProcessOrderJob.set(queue: :critical).perform_later(order.id)
```

## Queues

Declare which queue a job belongs to:

```ruby
class ReportJob < ApplicationJob
  queue_as :reports
end
```

Use a block when the queue name depends on arguments:

```ruby
class NotificationJob < ApplicationJob
  queue_as do
    arguments.first.premium? ? :premium : :default
  end
end
```

## Choose a Queue Backend

Set the adapter in `config/application.rb` (all environments) or per-environment:

```ruby
# config/application.rb
config.active_job.queue_adapter = :solid_queue   # Rails 8 default
config.active_job.queue_adapter = :sidekiq
config.active_job.queue_adapter = :good_job
config.active_job.queue_adapter = :inline        # synchronous — useful in dev
config.active_job.queue_adapter = :test          # captures jobs — use in tests
```

## Solid Queue (Rails 8 Default)

Solid Queue stores jobs in your primary database. No Redis required.

**Install:**

```bash
bin/rails solid_queue:install
bin/rails db:migrate
```

This creates `config/queue.yml` and a set of database tables (`solid_queue_jobs`, `solid_queue_processes`, etc.).

**Configure queues and concurrency** in `config/queue.yml`:

```yaml
# config/queue.yml
default: &default
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: "*"
      threads: 5
      polling_interval: 0.1

development:
  <<: *default

production:
  <<: *default
  workers:
    - queues: "critical,default"
      threads: 10
      polling_interval: 0.1
    - queues: "low"
      threads: 3
      polling_interval: 5
```

**Start workers** (Rails 8 Puma plugin handles this automatically in development):

```bash
bin/jobs          # starts Solid Queue worker in foreground
```

In production with Kamal, Solid Queue runs inside the same container as your web process when you use the Puma plugin:

```ruby
# config/puma.rb
plugin :solid_queue
```

## Sidekiq

Redis-backed, battle-tested at high throughput.

**Gemfile:**

```ruby
gem "sidekiq"
```

**Configure:**

```ruby
# config/application.rb
config.active_job.queue_adapter = :sidekiq
```

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV["REDIS_URL"] }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV["REDIS_URL"] }
end
```

**Start worker:**

```bash
bundle exec sidekiq
```

**Web UI** — mount in `config/routes.rb`:

```ruby
require "sidekiq/web"

Rails.application.routes.draw do
  authenticate :user, ->(u) { u.admin? } do
    mount Sidekiq::Web => "/sidekiq"
  end
end
```

**Pros:** Mature ecosystem, high throughput, rich UI, great observability.
**Cons:** Requires Redis, separate process to manage, jobs lost on Redis crash without persistence enabled.

## GoodJob

PostgreSQL-backed alternative to Sidekiq; no Redis, excellent visibility.

**Gemfile:**

```ruby
gem "good_job"
```

**Configure:**

```ruby
# config/application.rb
config.active_job.queue_adapter = :good_job
```

```ruby
# config/initializers/good_job.rb
GoodJob.configure do |config|
  config.max_threads = 10
  config.queues = "critical:5;default:3;low:1"
  config.poll_interval = 5
end
```

Mount the dashboard:

```ruby
# config/routes.rb
authenticate :user, ->(u) { u.admin? } do
  mount GoodJob::Engine => "/good_job"
end
```

## Retries and Error Handling

```ruby
class ImportJob < ApplicationJob
  queue_as :default

  # Retry on transient errors — exponential backoff by default
  retry_on Net::TimeoutError, wait: :polynomially_longer, attempts: 5

  # Discard jobs that fail due to data problems (no retry)
  discard_on ActiveRecord::RecordNotFound

  def perform(import_id)
    Import.find(import_id).process!
  end
end
```

`retry_on` options:

| Option | Effect |
|---|---|
| `wait: 5.seconds` | Fixed delay between attempts |
| `wait: :exponentially_longer` | 2^attempt seconds |
| `wait: :polynomially_longer` | attempt^4 seconds |
| `attempts: 3` | Max retry count (default: 5) |

## Best Practices

- **Pass IDs, not objects.** `perform(user.id)` not `perform(user)`.
- **Make jobs idempotent.** Running a job twice should produce the same result as running it once. Use `find_or_create_by`, guard clauses, and database unique constraints.
- **Keep jobs small.** Offload heavy logic to service objects; the job is just a thin wrapper.
- **Set timeouts.** Long-running jobs can block queues. Use database or HTTP timeouts inside `perform`.
- **Monitor queue depth.** Alert when jobs pile up — it signals a worker or dependency problem.

## Testing

Use `queue_adapter: :test` in `config/environments/test.rb` (Rails sets this automatically).

```ruby
# test/jobs/process_order_job_test.rb
require "test_helper"

class ProcessOrderJobTest < ActiveJob::TestCase
  test "processes the order" do
    order = orders(:pending)
    ProcessOrderJob.perform_now(order.id)
    assert order.reload.processed?
  end

  test "is enqueued with correct arguments" do
    order = orders(:pending)
    assert_enqueued_with(job: ProcessOrderJob, args: [order.id]) do
      ProcessOrderJob.perform_later(order.id)
    end
  end
end
```

Assert on enqueued jobs in controller or service tests:

```ruby
test "enqueues job on order creation" do
  assert_enqueued_with(job: ProcessOrderJob) do
    post orders_path, params: { order: { product_id: 1 } }
  end
end
```

Run all enqueued jobs inline inside a block:

```ruby
perform_enqueued_jobs do
  ProcessOrderJob.perform_later(order.id)
end
assert order.reload.processed?
```
