# Performance Issues

Use this file when your Rails application is correct but too slow. Find the symptom in the index, follow the decision tree, measure before and after every change.

## Symptom Index

| Symptom | Jump to |
|---|---|
| Individual page or endpoint is slow | [Slow Pages](#slow-pages) |
| High memory / server running out of RAM | [High Memory](#high-memory) |
| Background jobs are slow or backed up | [Slow Jobs](#slow-jobs) |
| Database queries are slow | [Database Bottlenecks](#database-bottlenecks) |

## Golden Rule

**Measure first.** Optimise only after you have data showing where the time is spent. Guessing leads to wasted effort and sometimes slower code.

```bash
# Read the log for a slow request — time is at the end of each request line
# "Completed 200 OK in 4823ms (Views: 312.4ms | ActiveRecord: 4201.1ms)"
tail -f log/development.log | grep "Completed"
```

---

## Slow Pages

### Step 1 — Read the request log

Every Rails request log line ends with a timing breakdown:

```
Completed 200 OK in 4823ms (Views: 312.4ms | ActiveRecord: 4201.1ms | Allocations: 94821)
```

| Component | High value → check |
|---|---|
| ActiveRecord time is high | SQL queries — go to [Database Bottlenecks](#database-bottlenecks) |
| Views time is high | Template rendering — go to [View Rendering](#view-rendering) |
| Total time high but both components low | External API call or cache miss |

### Step 2 — Decision tree

```
Slow page (> 500ms total)
  │
  ├─ ActiveRecord time > 80% of total?
  │     YES → go to Database Bottlenecks section
  │
  ├─ View time > 300ms?
  │     YES → go to View Rendering section
  │
  ├─ Total time high but DB + View time low?
  │     YES → External API call is blocking the response.
  │           Check: grep "Net::HTTP\|Faraday\|HTTParty" log/development.log
  │           Fix: move the API call to a background job.
  │                Cache the result with Rails.cache.fetch.
  │                Add a timeout to the HTTP client.
  │
  ├─ High Allocations count (> 500_000)?
  │     YES → Memory pressure / GC pauses adding latency.
  │           See High Memory section.
  │
  └─ Intermittent slowness (sometimes fast, sometimes slow)?
        Check: is the response cached? First request cold, subsequent warm?
        Check: is there a slow DB query on some data sizes but not others?
        Add: rack-mini-profiler for detailed per-request breakdown.
```

### rack-mini-profiler

Add to `Gemfile` (development group) for a per-request profiler overlay in the browser:

```ruby
# Gemfile
gem "rack-mini-profiler"
gem "flamegraph"       # optional — for flame graph views
gem "stackprof"        # optional — for flamegraph gem
gem "memory_profiler"  # optional — for memory profiling
```

```bash
bundle install
bin/rails server
# Visit any page — a timing badge appears in the top-left corner
```

---

## N+1 Queries

An N+1 query loads a collection of N records and then queries the database once per record to load an association.

**Symptom:** the log shows the same SELECT repeated many times:

```
SELECT "comments".* FROM "comments" WHERE "comments"."article_id" = 1
SELECT "comments".* FROM "comments" WHERE "comments"."article_id" = 2
SELECT "comments".* FROM "comments" WHERE "comments"."article_id" = 3
... (repeated for every article)
```

### Decision tree

```
N+1 queries
  │
  ├─ Identify the N+1
  │     Option 1: read the log — look for repeated SELECTs on the same table.
  │     Option 2: add bullet gem (development) — raises/logs N+1 warnings.
  │     Option 3: rack-mini-profiler — shows all queries per request.
  │
  ├─ Fix with eager loading
  │     Add includes() at the point the collection is loaded:
  │       Before: Article.all
  │       After:  Article.includes(:comments, :author)
  │
  ├─ preload vs eager_load vs includes
  │     includes:    Rails decides (preload or eager_load).
  │     preload:     always two queries (SELECT articles; SELECT comments WHERE id IN (...))
  │     eager_load:  always a JOIN (one query, larger result set)
  │     Use eager_load when you need to filter/order by the association:
  │       Article.eager_load(:author).where(authors: { active: true })
  │
  └─ Still N+1 after includes?
        Check: are you calling additional associations inside the loop?
               articles.each { |a| a.author.organisation }  # 3 levels deep
        Fix:   Article.includes(author: :organisation)
```

### bullet gem configuration

```ruby
# Gemfile
gem "bullet", group: :development

# config/environments/development.rb
config.after_initialize do
  Bullet.enable        = true
  Bullet.alert         = true
  Bullet.rails_logger  = true
  Bullet.add_footer    = true
end
```

---

## View Rendering

### Decision tree

```
High view render time
  │
  ├─ Is the view rendering a large collection without pagination?
  │     Check: how many records are in the collection?
  │     Fix:   add pagination (pagy gem — fastest, kaminari, will_paginate).
  │
  ├─ Are partials rendered in a loop?
  │     Fix:   use collection rendering (Rails caches the partial:
  │              render partial: "article", collection: @articles, cached: true
  │
  ├─ Are expensive computations happening in the view?
  │     Fix:   move to a presenter/decorator or the controller action.
  │            Cache the result with fragment caching.
  │
  ├─ Fragment caching?
  │     Rails.cache.fetch is not set up / not being used.
  │     Fix:   wrap slow view sections in cache blocks:
  │              <% cache @article do %>
  │                ... expensive partial ...
  │              <% end %>
  │     Ensure config.action_controller.perform_caching = true in development
  │     and the cache store is configured (Redis recommended in production).
  │
  └─ Asset loading slow?
        This shows as browser network time, not Rails View time.
        Fix:   enable CDN, gzip, HTTP/2, or asset fingerprinting.
```

### Fragment caching setup

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  expires_in: 1.hour
}

# config/environments/development.rb
config.action_controller.perform_caching = true
config.cache_store = :memory_store
```

```erb
<%# app/views/articles/index.html.erb %>
<% @articles.each do |article| %>
  <% cache article do %>
    <%= render article %>
  <% end %>
<% end %>
```

---

## High Memory

Rails processes grow over time due to large dataset loading, memory leaks in gems, or allocating objects unnecessarily.

### Step 1 — Measure baseline

```bash
# Check current memory usage of Puma workers
ps aux | grep puma | awk '{print $2, $6/1024 "MB", $11}'

# In production — watch RSS over time
watch -n5 'ps aux | grep puma'
```

### Decision tree

```
High memory / memory growing over time
  │
  ├─ Are large datasets loaded into memory at once?
  │     Check: Model.all, Model.where(...) with thousands of records.
  │     Fix:   use find_each (batches of 1000) instead of all.each:
  │              Before: User.all.each { |u| process(u) }
  │              After:  User.find_each { |u| process(u) }
  │            For arbitrary queries: in_batches(of: 500).each_record
  │
  ├─ Memory growing steadily (leak)?
  │     Tool: derailed_benchmarks gem
  │       bundle exec derailed bundle:mem    # which gems use most memory
  │       bundle exec derailed exec perf:mem # memory per request
  │     Tool: memory_profiler
  │       require 'memory_profiler'
  │       report = MemoryProfiler.report { your_code }
  │       report.pretty_print
  │
  ├─ Large file uploads or downloads held in memory?
  │     Fix:   stream files instead of loading them:
  │              send_file path, disposition: "attachment"  # does not buffer
  │            For S3/cloud storage: redirect_to presigned_url  (zero memory)
  │
  ├─ Caching too much in memory store?
  │     Fix:   switch from :memory_store to :redis_cache_store in production.
  │
  └─ Jemalloc not installed?
        Jemalloc reduces Ruby heap fragmentation.
        Fix:   install libjemalloc and set LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2
               Many production Ruby Docker images include it.
```

### find_each and in_batches

```ruby
# find_each — simplest batching
User.where(active: true).find_each(batch_size: 500) do |user|
  UserMailer.newsletter(user).deliver_later
end

# in_batches — when you need the batch as a relation
User.where(active: true).in_batches(of: 500) do |batch|
  batch.update_all(last_newsletter_sent_at: Time.current)
end
```

### derailed_benchmarks

```ruby
# Gemfile (development group)
gem "derailed_benchmarks", require: false

# Measure which gems allocate the most memory
bundle exec derailed bundle:mem

# Run a specific path 100 times and report memory delta
TEST_COUNT=100 bundle exec derailed exec perf:mem_over_time
```

---

## Slow Jobs

### Step 1 — Check queue depth

A large queue depth means jobs are backing up faster than workers can process them.

```bash
# Sidekiq — open the web UI (usually /sidekiq after mounting)
# Or via console:
bin/rails runner "puts Sidekiq::Queue.new.size"
bin/rails runner "puts Sidekiq::Stats.new.to_h"

# Good Activejob adapter info:
bin/rails runner "puts ActiveJob::QueueAdapters::SidekiqAdapter.queue_size('default')"
```

### Decision tree

```
Slow or backed-up background jobs
  │
  ├─ Queue depth growing (more jobs enqueued than processed)?
  │     Fix:   add more workers / increase concurrency:
  │            Sidekiq: increase concurrency in config/sidekiq.yml (default 10)
  │            or run additional Sidekiq processes / dynos / containers.
  │     Check: is the bottleneck CPU, I/O, or DB connections?
  │            CPU-bound → more processes (not threads).
  │            I/O-bound → more threads is fine.
  │            DB connections → increase connection pool AND DB max_connections.
  │
  ├─ Individual job duration too long?
  │     Instrument: add ActiveSupport::Notifications or manual timing:
  │       start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
  │       # ... do work ...
  │       elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
  │       Rails.logger.info "Job took #{elapsed.round(2)}s"
  │
  │     Is the job loading a large dataset?
  │       Fix: use find_each (see High Memory section).
  │
  │     Is the job making N+1 queries?
  │       Fix: use includes() (see N+1 Queries section).
  │
  │     Is the job calling external APIs in series?
  │       Fix: batch the API calls or parallelize with Thread / Concurrent::Future.
  │
  ├─ Jobs retrying frequently?
  │     Check: Sidekiq UI retry queue — what exceptions are causing retries?
  │     Fix:   fix the underlying error, or add discard_on for non-retriable errors.
  │
  └─ Scheduled/recurring jobs bunching up?
        Check: are jobs from the previous run still running when the next fires?
        Fix:   add a unique job lock (sidekiq-unique-jobs) or increase the interval.
```

### Sidekiq configuration

```yaml
# config/sidekiq.yml
:concurrency: 10
:queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]
:max_retries: 5
```

```ruby
# app/jobs/heavy_job.rb
class HeavyJob < ApplicationJob
  queue_as :low
  sidekiq_options retry: 3, dead: false

  discard_on ActiveRecord::RecordNotFound

  def perform(record_id)
    record = Record.find(record_id)
    # process record
  end
end
```

---

## Database Bottlenecks

### Step 1 — Enable the slow query log

**PostgreSQL** — log queries longer than 100ms:

```sql
-- In psql, or add to postgresql.conf:
ALTER SYSTEM SET log_min_duration_statement = '100ms';
SELECT pg_reload_conf();
```

**MySQL / MariaDB** — add to my.cnf:

```ini
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 0.1
```

**SQLite (development):**

```ruby
# config/environments/development.rb
config.log_level = :debug  # shows all SQL
```

### Step 2 — EXPLAIN a slow query

```sql
-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM articles WHERE author_id = 42 ORDER BY created_at DESC LIMIT 20;

-- MySQL
EXPLAIN SELECT * FROM articles WHERE author_id = 42 ORDER BY created_at DESC LIMIT 20;
```

**Red flags in EXPLAIN output:**

| Flag | Meaning | Fix |
|---|---|---|
| Seq Scan on large table | No index used | Add an index |
| rows estimate >> actual rows | Outdated statistics | ANALYZE the table |
| Nested Loop with many rows | N+1 at the DB level | Rewrite the query |
| Hash / Sort on disk | Memory spilled to disk | Increase work_mem (PostgreSQL) |

### Decision tree

```
Database bottleneck
  │
  ├─ Missing index?
  │     Check: EXPLAIN output shows Seq Scan on a table with > 10k rows.
  │     Check: query has WHERE, ORDER BY, or JOIN on unindexed columns.
  │     Fix:   add an index via migration:
  │              add_index :articles, :author_id
  │              add_index :articles, [:status, :created_at]  # composite
  │     Verify: run EXPLAIN again — should show Index Scan.
  │
  ├─ N+1 queries at the DB level?
  │     See N+1 Queries section above.
  │
  ├─ Missing composite index?
  │     Example: WHERE status = 'active' ORDER BY created_at DESC
  │     A single index on status or created_at alone may not be used.
  │     Fix:   add_index :articles, [:status, :created_at]
  │            Column order matters: put equality conditions first.
  │
  ├─ Table too large — queries slow despite indexes?
  │     Options:
  │     - Partition the table (PostgreSQL native partitioning)
  │     - Archive old rows to a separate table
  │     - Add a database read replica and route read queries to it
  │
  ├─ Slow full-text search (LIKE '%keyword%')?
  │     LIKE with a leading wildcard cannot use a B-tree index.
  │     Fix (PostgreSQL):  use pg_search gem (tsvector / GIN index).
  │     Fix (MySQL):       use FULLTEXT index and MATCH AGAINST.
  │     Fix (any):         use a dedicated search service (Elasticsearch, Typesense, Meilisearch).
  │
  └─ Connection pool exhausted?
        Check: your APM or db server pg_stat_activity.
        Fix:   increase pool in database.yml (but stay within DB max_connections).
               Use PgBouncer in transaction pooling mode for high-concurrency apps.
```

### Adding indexes via migration

```bash
bin/rails generate migration AddIndexToArticlesAuthorId
```

```ruby
class AddIndexToArticlesAuthorId < ActiveRecord::Migration[8.0]
  def change
    # Simple index
    add_index :articles, :author_id

    # Composite index for common query patterns
    add_index :articles, [:status, :created_at]

    # Partial index (PostgreSQL) — only index published articles
    add_index :articles, :created_at, where: "status = 'published'"

    # Concurrent index (PostgreSQL) — no table lock
    add_index :articles, :slug, unique: true, algorithm: :concurrently
  end
end
```

### pgBadger / pg_activity (PostgreSQL)

```bash
# pg_activity — live view of running queries
pip install pg_activity
pg_activity -U postgres

# pgBadger — parse log file and generate HTML report
pgbadger /var/log/postgresql/postgresql-*.log -o report.html
open report.html
```

---

## Caching Strategy

Caching is the highest-leverage performance tool once you have eliminated N+1 queries and missing indexes.

### Cache layers in Rails

| Layer | Tool | Best for |
|---|---|---|
| HTTP caching | `stale?`, `fresh_when`, `expires_in` | Repeated identical requests |
| Fragment caching | `cache` helper in views | Expensive view partials |
| Low-level caching | `Rails.cache.fetch` | Computed values, external API responses |
| Russian doll caching | Nested `cache` blocks | Collections with dependent records |
| Whole-page caching | `ActionController::Caching::Pages` | Fully public, rarely changing pages |

### Low-level cache pattern

```ruby
# Cache an expensive computation for 10 minutes
def popular_articles
  Rails.cache.fetch("popular_articles", expires_in: 10.minutes) do
    Article.published
           .joins(:views)
           .group("articles.id")
           .order("COUNT(views.id) DESC")
           .limit(10)
           .to_a   # force evaluation before caching
  end
end
```

### HTTP caching in controllers

```ruby
def show
  @article = Article.find(params[:id])
  # Return 304 Not Modified if the client's cached version is still fresh
  fresh_when @article, public: true
end

def index
  @articles = Article.published.order(created_at: :desc)
  expires_in 5.minutes, public: true
end
```

---

## Performance Checklist

Use this checklist before declaring a performance investigation complete:

```
[ ] Request log shows timing breakdown (ActiveRecord vs View)
[ ] No N+1 queries (bullet gem clean, or queries reviewed manually)
[ ] Slow queries identified with slow query log or EXPLAIN
[ ] Indexes added for all WHERE, JOIN, ORDER BY columns on large tables
[ ] Large datasets use find_each / in_batches (never .all.each)
[ ] Fragment or low-level caching applied to expensive computed sections
[ ] External API calls moved to background jobs or cached
[ ] Memory usage stable over time (no leak confirmed with derailed_benchmarks)
[ ] Response times measured before and after each change
```
