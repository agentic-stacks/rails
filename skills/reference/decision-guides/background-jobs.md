# Background Jobs Decision Guide

Choose between Solid Queue, Sidekiq, and GoodJob for your Rails app.

---

## Comparison Table

| Criterion | Solid Queue | Sidekiq | GoodJob |
|---|---|---|---|
| **Rails 8 default** | Yes | No | No |
| **Backend / storage** | Database (any adapter) | Redis | PostgreSQL only |
| **External dependency** | None | Redis | None (uses app DB) |
| **Web UI** | Via Mission Control | Yes (Sidekiq Web) | Yes (GoodJob Dashboard) |
| **Recurring jobs** | Yes (via `solid_queue` config) | Via `sidekiq-cron` gem | Yes (native) |
| **Concurrency model** | Thread per worker | Thread per worker | Thread per worker |
| **Multi-tenancy support** | Limited | Via gems | Excellent |
| **Job priority** | Yes | Yes | Yes |
| **Job pausing** | Yes | Via Sidekiq Pro | Yes |
| **Bulk enqueue** | Yes | Yes (batch, Pro) | Limited |
| **ActiveJob integration** | Yes | Yes | Yes |
| **Native API (non-AJ)** | No | Yes (`Sidekiq::Worker`) | Yes (`GoodJob::Job`) |
| **Maturity** | Emerging (2024) | Very mature (2012) | Mature (2020) |
| **Cost** | Free | Free / Pro ($) | Free |
| **Throughput (jobs/sec)** | Medium | Very high | Medium |
| **Rails 7.x gem** | Available (`solid_queue` gem) | Available | Available |

---

## Option Details

### Solid Queue (Rails 8 Default)

Solid Queue is a database-backed queue built for Rails. It uses your existing database (SQLite, PostgreSQL, MySQL) to store and dispatch jobs. No additional infrastructure is required.

**Strengths:**
- Zero infrastructure overhead — no Redis, no separate service to operate
- Native Rails integration: included in Rails 8, configured by default
- Runs inside the Puma process in development (via Puma plugin) — one command to start everything
- Works with SQLite in development; scales to PostgreSQL in production
- Atomic job dispatch using `SELECT ... FOR UPDATE SKIP LOCKED` — battle-tested pattern

**Weaknesses:**
- Throughput is bounded by database write capacity; not suitable for very high-throughput queues (thousands of jobs/second)
- Less mature ecosystem compared to Sidekiq — fewer plugins and integrations
- Web UI requires the separate Mission Control gem
- Not a drop-in for existing Sidekiq-heavy workflows (no Sidekiq-specific APIs)

**Solid Queue web UI** (Mission Control):
```ruby
# Gemfile
gem "mission_control-jobs"
```
```ruby
# config/routes.rb
authenticate :user, ->(u) { u.admin? } do
  mount MissionControl::Jobs::Engine, at: "/jobs"
end
```

**When to avoid:**
- Apps with very high job throughput (> 1,000 jobs/sec sustained)
- Teams already standardised on Sidekiq across multiple services
- Apps that need Redis for other reasons (e.g., Action Cable) — if Redis is already there, Sidekiq adds little overhead

---

### Sidekiq

Redis-backed job queue. The de facto standard for Ruby background processing. Extremely high throughput, mature ecosystem, widely deployed.

**Strengths:**
- Battle-tested since 2012; used in production by thousands of companies
- Very high throughput — millions of jobs per day on a single server
- Rich ecosystem: `sidekiq-cron`, `sidekiq-unique-jobs`, `sidekiq-status`, etc.
- Excellent built-in web UI with live queue visibility, job retry, and dead job management
- Pro and Enterprise tiers add batch jobs, rate limiting, encryption, and multi-tenant support

**Weaknesses:**
- Requires Redis — an additional service to provision, monitor, and keep available
- Redis persistence must be configured explicitly (`appendonly yes`) or jobs are lost on Redis restart
- Separate process to start (`bundle exec sidekiq`)
- Sidekiq Pro/Enterprise features cost money for non-OSS projects

**Cost:**
- Sidekiq (OSS): Free
- Sidekiq Pro: ~$395/app/year
- Sidekiq Enterprise: ~$1,495/app/year

**When to avoid:**
- Apps where Redis would be introduced solely for Sidekiq — consider Solid Queue or GoodJob instead
- Small apps or hobby projects where infrastructure cost matters

---

### GoodJob

PostgreSQL-backed alternative to Sidekiq. Excellent observability, native recurring jobs, and Rails-idiomatic configuration. No Redis required.

**Strengths:**
- Uses PostgreSQL's `FOR UPDATE SKIP LOCKED` for reliable job locking
- Built-in web dashboard with live job monitoring, filtering, and retry
- Native cron/recurring job support without extra gems
- Excellent multi-tenancy support (compatible with `apartment`, `acts_as_tenant`)
- Jobs are durable — stored in the database, recovered on restart
- Well-maintained with active development

**Weaknesses:**
- PostgreSQL only — cannot use with MySQL or SQLite
- Throughput is lower than Sidekiq for high-volume workloads
- Adds tables to the application database (manageable but worth noting)
- Smaller ecosystem than Sidekiq

**When to avoid:**
- Apps using MySQL or SQLite as the primary database
- Apps that need very high throughput (Sidekiq is more appropriate)

---

## Recommendations by Use Case

| Use Case | Recommended | Rationale |
|---|---|---|
| New Rails 8 app, no Redis, low–medium job volume | Solid Queue | Zero infrastructure, matches Rails 8 defaults |
| New Rails 8 app, PostgreSQL, needs recurring jobs / rich UI | GoodJob | Native cron, excellent dashboard, no Redis |
| High-throughput production app (> 10k jobs/day sustained) | Sidekiq | Proven throughput, rich ecosystem |
| Existing app already using Sidekiq | Keep Sidekiq | Migration cost is rarely justified unless Redis is being removed |
| App with multi-tenant architecture on PostgreSQL | GoodJob | Best multi-tenancy support |
| App that must eliminate external infrastructure dependencies | Solid Queue or GoodJob | Both require only the app database |
| App needing Sidekiq-specific features (batches, unique jobs) | Sidekiq Pro | No equivalent in Solid Queue or GoodJob |

---

## Switching Queue Backends

All three integrate with Active Job. Swapping backends requires:

1. Update `config/application.rb`:
   ```ruby
   config.active_job.queue_adapter = :solid_queue  # or :sidekiq, :good_job
   ```

2. Install the gem and run its install generator:
   ```bash
   # Solid Queue
   bin/rails solid_queue:install && bin/rails db:migrate

   # GoodJob
   bin/rails generate good_job:install && bin/rails db:migrate

   # Sidekiq — no generator; configure via initializer
   ```

3. Drain the old queue before switching in production. Do not switch adapters with jobs in-flight — they will be orphaned.

4. Update `Procfile.dev` and production process management to start the new worker.
