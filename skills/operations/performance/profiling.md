# Profiling

Identify exactly where time and memory go before changing any code.

## rack-mini-profiler

Add to `Gemfile`:

```ruby
group :development do
  gem "rack-mini-profiler"
  gem "flamegraph"      # enables flamegraph drilldown
  gem "stackprof"       # required for flamegraph
end
```

```bash
bundle install
```

No other configuration needed. A timing badge appears in the top-left corner of every HTML response. Click it to see a breakdown of time by SQL query, cache call, and rendering.

### Key URLs

```
/?pp=flamegraph        # CPU flamegraph of the current request
/?pp=profile-memory    # memory allocation breakdown
/?pp=analyze-memory    # object count by type
/?pp=stackprof         # stackprof CPU profile
```

### Enable in Staging / Production (with auth)

```ruby
# config/initializers/rack_mini_profiler.rb
if Rails.env.staging?
  Rack::MiniProfiler.config.pre_authorize_cb = lambda do |env|
    env["warden"]&.user&.admin?
  end
end
```

## Benchmark Module

Use the standard library `Benchmark` to time specific code paths.

```ruby
require "benchmark"

# Single measurement
time = Benchmark.measure do
  User.includes(:posts).where(active: true).to_a
end
puts time.real  # wall clock seconds

# Compare multiple approaches
Benchmark.bm(30) do |x|
  x.report("includes:") { 100.times { User.includes(:posts).first(10) } }
  x.report("preload:")  { 100.times { User.preload(:posts).first(10) } }
  x.report("joins:")    { 100.times { User.joins(:posts).first(10) } }
end

# Benchmark.bmbm runs each twice (rehearsal then real) to warm up allocator
Benchmark.bmbm do |x|
  x.report("with index:")    { 1000.times { User.find_by(email: "a@b.com") } }
  x.report("without index:") { 1000.times { User.find_by(name: "Alice") } }
end
```

## memory_profiler

Track object allocations and retained memory.

```ruby
group :development, :test do
  gem "memory_profiler"
end
```

```ruby
require "memory_profiler"

report = MemoryProfiler.report do
  # code to profile
  Post.includes(:author, :tags).limit(100).to_a
end

report.pretty_print(to_file: "/tmp/memory_report.txt")
# Or to stdout:
report.pretty_print
```

Key sections in the report:

| Section | Meaning |
|---|---|
| Total allocated | All memory allocated during the block |
| Total retained | Memory still live after the block — these are leaks |
| Top allocators by gem | Which gems cause most allocations |
| Top allocators by file | Your own code's hotspots |

Aim to minimize **retained** objects. Allocated objects are normal; retained ones indicate leaks.

## derailed_benchmarks

Measure real-world memory at boot and under load without writing test code.

```ruby
group :development do
  gem "derailed_benchmarks"
end
```

```bash
# How much memory does booting Rails use?
bundle exec derailed bundle:mem

# Which gems contribute most to memory at boot?
bundle exec derailed bundle:objects

# Memory growth per request (run the app first)
bundle exec derailed exec perf:mem

# Object count per request
bundle exec derailed exec perf:objects
```

Set `TEST_COUNT` to control iterations:

```bash
TEST_COUNT=200 bundle exec derailed exec perf:mem
```

## stackprof

CPU sampling profiler. Low overhead — safe in production with care.

```ruby
group :development do
  gem "stackprof"
end
```

```ruby
StackProf.run(mode: :cpu, out: "/tmp/stackprof.dump", interval: 1000) do
  # code to profile
  PostSerializer.new(Post.includes(:author).limit(500).to_a).as_json
end
```

```bash
# Print a text report
stackprof /tmp/stackprof.dump --text --limit 20

# Print with source lines
stackprof /tmp/stackprof.dump --method "Post#as_json"
```

Rack middleware for continuous profiling:

```ruby
# config/environments/development.rb
config.middleware.use StackProf::Middleware,
  enabled: true,
  mode: :cpu,
  interval: 1000,
  save_every: 5   # save a dump file every 5 requests
```

## Watching Rails Logs

```bash
# Follow development log, highlight slow queries
tail -f log/development.log | grep --color -E "SLOW|[0-9]{3,}ms"

# Find all queries over 100ms in a log file
grep -E "\([0-9]{3,}\.[0-9]+ms\)" log/development.log | sort -t'(' -k2 -rn | head -20
```

Set a slow query threshold:

```ruby
# config/initializers/query_log.rb (Rails 7+)
ActiveSupport::Notifications.subscribe("sql.active_record") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  if event.duration > 100 # ms
    Rails.logger.warn "[SLOW QUERY #{event.duration.round}ms] #{event.payload[:sql]}"
  end
end
```

## Production APM

Use a production APM service to get continuous profiling, error tracking, and alerting without modifying code.

### Scout APM

```ruby
gem "scout_apm"
```

```bash
SCOUT_KEY=your_key SCOUT_NAME="MyApp" rails server
```

Auto-detects slow transactions, N+1 queries, and memory issues. Web UI shows traces with drill-down.

### Skylight

```ruby
gem "skylight"
```

```bash
bundle exec skylight setup   # interactive auth
```

Strong on showing aggregate endpoint performance and trend graphs. Rails-native instrumentation.

### New Relic

```ruby
gem "newrelic_rpm"
```

```bash
# Outputs config/newrelic.yml
bundle exec newrelic install --license_key=YOUR_KEY AppName
```

Enterprise-grade. Distributed tracing, custom dashboards, alerting on Apdex scores.

### What to monitor with any APM

- **Throughput** — requests/minute per endpoint
- **Response time percentiles** — p50, p95, p99 (median hides the tail)
- **Error rate** — spikes indicate regressions
- **Memory** — steady growth indicates a leak
- **Database time %** — if > 60% of response time is DB, focus there first
