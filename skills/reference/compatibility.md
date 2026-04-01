# Compatibility Reference

Ruby, Rails, gem, Node, and database adapter version compatibility.

---

## Ruby / Rails Version Matrix

| Rails | Min Ruby | Max Tested | Recommended | EOL (Rails) |
|---|---|---|---|---|
| 7.0 | 2.7 | 3.3 | 3.1+ | Jun 2025 |
| 7.1 | 2.7 | 3.3 | 3.2+ | Oct 2025 |
| 7.2 | 3.1 | 3.3 | 3.3+ | Aug 2026 |
| 8.0 | 3.2 | 3.4 | 3.3+ | Nov 2027 |

Notes:
- "Max Tested" reflects the Ruby version included in CI at time of release; newer Rubies are typically backward-compatible.
- Ruby 3.4 is supported experimentally in Rails 8.0.0 and fully from 8.0.2.
- Use the recommended Ruby version for new projects. It matches the version used in generated Dockerfiles.

---

## Ruby Version Compatibility Details

| Ruby | Status | Notes |
|---|---|---|
| 2.7 | EOL (Apr 2023) | Works with Rails 7.0/7.1 but unsupported; use only if constrained |
| 3.0 | EOL (Apr 2024) | Works with Rails 7.x; avoid for new projects |
| 3.1 | Security fixes until Mar 2025 | Minimum for Rails 7.2; stable and widely hosted |
| 3.2 | Supported | Minimum for Rails 8.0; YJIT enabled by default |
| 3.3 | Supported (current stable) | Recommended for all Rails 7.2+ and Rails 8 projects |
| 3.4 | Supported (current dev) | Supported in Rails 8.0.2+; some gems still catching up |

---

## Key Gem Compatibility

### ActiveRecord / Database Adapters

| Gem | Rails 7.0 | Rails 7.1 | Rails 7.2 | Rails 8.0 | Notes |
|---|---|---|---|---|---|
| `pg` (PostgreSQL) | 1.x | 1.x | 1.x | 1.x | Stable across all versions |
| `mysql2` | 0.5.x | 0.5.x | 0.5.x | 0.5.x | Stable; use `>= 0.5.5` |
| `sqlite3` | 1.4+ | 1.4+ | 1.6+ | 2.x | Rails 8 prefers `sqlite3 >= 2.0` for Solid* gems |
| `trilogy` | — | 0.x | 1.x | 1.x | MySQL-compatible; default in some Rails 8 configs |

### Authentication

| Gem | Rails 7.0 | Rails 7.1 | Rails 7.2 | Rails 8.0 | Notes |
|---|---|---|---|---|---|
| `devise` | 4.9+ | 4.9+ | 4.9+ | 4.9+ | Requires `devise >= 4.9.3` for Rails 7.1 token changes |
| `doorkeeper` | 5.6+ | 5.6+ | 5.6+ | 5.6+ | OAuth 2.0 provider |
| `omniauth` | 2.x | 2.x | 2.x | 2.x | |

### Background Jobs

| Gem | Rails 7.0 | Rails 7.1 | Rails 7.2 | Rails 8.0 | Notes |
|---|---|---|---|---|---|
| `solid_queue` | — | 0.x | 1.x | bundled | Ships with Rails 8; gem install required for 7.x |
| `sidekiq` | 6.x/7.x | 7.x | 7.x | 7.x | `sidekiq >= 7.0` required for Rails 7.1+ |
| `good_job` | 3.x | 3.x | 3.x/4.x | 4.x | PostgreSQL only; excellent for Rails 7.1+ |

### Asset Pipeline

| Gem | Rails 7.0 | Rails 7.1 | Rails 7.2 | Rails 8.0 | Notes |
|---|---|---|---|---|---|
| `sprockets-rails` | 3.4+ | 3.4+ | 3.4+ | not default | Maintained but not the Rails 8 default |
| `propshaft` | optional | optional | optional | default | Default in Rails 8; simpler than Sprockets |
| `importmap-rails` | 1.x | 1.x/2.x | 2.x | 2.x | `>= 2.0` required with Propshaft |
| `jsbundling-rails` | 1.x | 1.x | 1.x | 1.x | Required for esbuild/rollup/webpack with Propshaft |
| `cssbundling-rails` | 1.x | 1.x | 1.x | 1.x | Required for Tailwind CLI, PostCSS outside asset pipeline |

### Caching

| Gem | Rails 7.0 | Rails 7.1 | Rails 7.2 | Rails 8.0 | Notes |
|---|---|---|---|---|---|
| `solid_cache` | — | 0.x | 1.x | bundled | Ships with Rails 8 |
| `redis-rails` | deprecated | — | — | — | Use `redis-actionpack` directly or switch to Solid Cache |

---

## Node.js Version Requirements

Node is only required when using a JavaScript bundler (esbuild, rollup, webpack, Bun). It is not required for importmap-only apps.

| Bundler | Min Node | Recommended Node | Notes |
|---|---|---|---|
| `esbuild` (via jsbundling-rails) | 14 | 20 LTS | Node 16 EOL; use 18+ for CI |
| `rollup` | 14 | 20 LTS | |
| `webpack` (via webpacker, legacy) | 12 | 18 LTS | Webpacker is unmaintained; migrate away |
| Bun | — | 1.x | Replaces Node entirely; supported from Rails 7.1 |
| Tailwind CSS CLI | 16 | 20 LTS | Tailwind v4 requires Node 18+ |

Node LTS schedule:
- Node 16: EOL Sep 2023 — do not use
- Node 18: LTS until Apr 2025
- Node 20: LTS until Apr 2026 (recommended)
- Node 22: LTS from Oct 2024

---

## Database Adapter Compatibility

### PostgreSQL

| pg gem | PostgreSQL server | Notes |
|---|---|---|
| 1.5.x | 9.4 – 16 | Recommended; bundled OpenSSL |
| 1.4.x | 9.4 – 15 | Stable; lacks some connection retry options |

Rails 8 requires PostgreSQL 9.3+ for `jsonb` support in Solid Queue migrations.

### MySQL / MariaDB

| mysql2 gem | MySQL | MariaDB | Notes |
|---|---|---|---|
| 0.5.5+ | 5.7 – 8.0+ | 10.3 – 11.x | Use utf8mb4 encoding; avoid utf8 (3-byte alias) |

Solid Queue MySQL migration issue: see `known-issues/rails-8.md`.

### SQLite

| sqlite3 gem | SQLite | Notes |
|---|---|---|
| 2.x | 3.38+ | Required for Rails 8 Solid* gems; WAL mode default |
| 1.6.x | 3.28+ | Compatible with Rails 7.x |
| 1.4.x | 3.16+ | Minimum for Rails 7.0; lacks advisory lock support |

SQLite WAL mode (enabled by default in `sqlite3 >= 2.0`) allows concurrent reads. Recommended for development and low-traffic production.

---

## Bundler and RubyGems

| Rails | Min Bundler | Recommended Bundler |
|---|---|---|
| 7.0 | 2.2 | 2.4+ |
| 7.1 | 2.3 | 2.5+ |
| 7.2 | 2.3 | 2.5+ |
| 8.0 | 2.4 | 2.6+ |

Always run `gem update bundler` when setting up a new environment.
