# Diagnose

This skill group covers diagnosing problems in Rails 7.x and 8.x applications. Use the symptom index below to navigate to the right file, then follow the decision tree to identify and fix the issue.

## Symptom Index

| Symptom | Start here |
|---|---|
| App won't start / crashes on boot | [boot-failures.md](boot-failures.md) |
| Specific error message in logs or browser | [common-errors.md](common-errors.md) |
| HTTP 4xx or 5xx response | [request-errors.md](request-errors.md) |
| Slow pages, slow queries, high memory, slow jobs | [performance-issues.md](performance-issues.md) |

## How to Use These Files

Each file follows a **symptom → check → fix** pattern:

1. **Symptom** — what you observe (crash, error class, status code, latency spike).
2. **Check** — the smallest command or inspection that confirms the cause.
3. **Fix** — the concrete change that resolves it.
4. **Prevent** — what to add so the problem does not recur.

Decision trees branch on the result of each check. Work top-to-bottom. Most branches include a verification step so you know when you are done.

## General Debugging Workflow

Before diving into a specific file, collect context:

```bash
# 1. What does the log say?
tail -f log/development.log

# 2. Which Rails version?
bin/rails --version

# 3. Which Ruby version?
ruby --version

# 4. Are there pending migrations?
bin/rails db:migrate:status

# 5. Are all gems installed?
bundle check
```

For production incidents, always capture the full backtrace from your error tracker (Sentry, Honeybadger, Rollbar, Bugsnag) before changing anything. The first frame that is your application code — not a framework file — is almost always the root cause.

## Files in This Skill

| File | What it covers |
|---|---|
| `boot-failures.md` | App crashes before it can serve any request |
| `common-errors.md` | Top 20 Rails exception classes with decision trees |
| `request-errors.md` | HTTP 400/401/403/404/422/500/502/503/504 diagnosis |
| `performance-issues.md` | Slow pages, N+1 queries, high memory, slow jobs, DB bottlenecks |
