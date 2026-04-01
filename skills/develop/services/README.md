# Services

Rails ships with a set of service subsystems that handle work beyond the request-response cycle: queuing background jobs, caching computed data, sending email, and managing file attachments. Each subsystem has a standard interface and pluggable backends.

## Subsystems at a Glance

| Subsystem | Rails component | Default backend (Rails 8) |
|---|---|---|
| Background jobs | Active Job | Solid Queue |
| Caching | Rails.cache / fragment cache | Solid Cache |
| Email | Action Mailer | SMTP / async via Active Job |
| File attachments | Active Storage | Local disk (dev), S3/GCS/Azure (prod) |

## Decision Tree

```
What do you need?
├── Run code outside the request cycle
│     └── background-jobs.md
├── Avoid re-computing expensive data on every request
│     └── caching.md
├── Send email
│     └── mailer.md
└── Store and serve user-uploaded files
      └── active-storage.md
```

## Quick-Install Reference (Rails 8)

Install all service backends in one pass for a new app:

```bash
# Background jobs (database-backed queue)
bin/rails solid_queue:install

# Caching (database-backed cache store)
bin/rails solid_cache:install

# Active Storage (local disk by default)
bin/rails active_storage:install

# Run all pending migrations
bin/rails db:migrate
```

## How the Subsystems Relate

```
HTTP Request
  └─► Controller
        ├─► UserMailer.welcome(user).deliver_later
        │     └─► Active Job ──► Solid Queue / Sidekiq / GoodJob
        │
        ├─► Rails.cache.fetch("dashboard/#{user.id}") { ... }
        │     └─► Solid Cache / Redis / Memcached
        │
        └─► user.avatar.attach(params[:avatar])
              └─► Active Storage ──► Local disk / S3 / GCS
```

## Files in This Skill

| File | What it covers |
|---|---|
| `background-jobs.md` | Active Job, Solid Queue, Sidekiq, GoodJob, retries, testing |
| `caching.md` | Fragment caching, Russian doll, low-level cache, Solid Cache, Redis |
| `mailer.md` | Action Mailer, previews, delivery methods, interceptors, attachments |
| `active-storage.md` | File uploads, variants, direct upload, storage service config |

## Next Steps

- Add asynchronous work to your app: `background-jobs.md`
- Speed up slow pages: `caching.md`
- Send transactional email: `mailer.md`
- Accept file uploads: `active-storage.md`
