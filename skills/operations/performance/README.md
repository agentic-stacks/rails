# Performance

Performance work follows one rule: measure before you optimize. Guessing at bottlenecks wastes time and can make things worse.

## Measure First

```bash
# Enable query logging in development (already on by default)
# Check log/development.log for slow queries

# Time any block in the console
bin/rails console
> require 'benchmark'
> Benchmark.measure { 1000.times { User.where(active: true).to_a } }.real
```

Check your Rails logs for the two numbers that matter most:
- **Completed in Xms** — total request time
- **Views: Xms | ActiveRecord: Xms** — where the time went

## Common Bottlenecks (in order of frequency)

| Bottleneck | Symptom | File to read |
|---|---|---|
| N+1 queries | Many small identical queries in logs | `n-plus-one.md` |
| Missing indexes | Seq Scan in EXPLAIN output, slow `WHERE`/`ORDER BY` | `database-tuning.md` |
| No caching | Same data fetched repeatedly per request | `caching-strategy.md` |
| Slow external APIs | High total time, low ActiveRecord time | `profiling.md` |
| Memory bloat | High memory, GC pauses, slow response over time | `profiling.md` |
| Unoptimized queries | Table scans, loading unused columns, large result sets | `database-tuning.md` |

## Quick Wins Decision Tree

```
App is slow — where do I start?
│
├── Is it slow only in production, not development?
│   ├── Yes — check APM (Scout / Skylight / New Relic)  → profiling.md
│   └── No  — reproduce locally, see below
│
├── Open Rails logs — how many queries per request?
│   ├── Dozens of similar queries → N+1 problem         → n-plus-one.md
│   └── Few queries, each fast   → not a DB problem
│
├── Are there slow queries (> 100ms each)?
│   ├── Yes — run EXPLAIN ANALYZE on the query          → database-tuning.md
│   └── No  — DB is not the bottleneck
│
├── Is the view render time high?
│   ├── Yes — fragment caching, fewer partials          → caching-strategy.md
│   └── No  — look elsewhere
│
├── Is memory growing per request / over time?
│   ├── Yes — memory_profiler / derailed_benchmarks     → profiling.md
│   └── No
│
└── Is the bottleneck an external service call?
    ├── Yes — add caching, use async jobs               → caching-strategy.md
    └── No  — use rack-mini-profiler to drill down      → profiling.md
```

## Files in This Skill

| File | What It Covers |
|---|---|
| `profiling.md` | rack-mini-profiler, Benchmark, memory_profiler, stackprof, APM |
| `n-plus-one.md` | Detecting and fixing N+1 queries, Bullet, strict_loading |
| `caching-strategy.md` | Cache layers, invalidation patterns, when not to cache |
| `database-tuning.md` | Indexes, EXPLAIN, query optimization, connection pooling |

## Golden Rules

1. **Profile first** — never optimize blind.
2. **Fix N+1 before adding caching** — cache a broken query and you cache a problem.
3. **Add indexes before rewriting queries** — a well-indexed table makes most queries fast.
4. **Cache at the highest useful layer** — HTTP cache beats fragment cache beats low-level cache.
5. **Monitor in production** — development performance does not equal production performance.
