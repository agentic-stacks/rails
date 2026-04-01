# Advanced Rails Patterns

These patterns solve real, specific problems. Each one adds meaningful complexity — extra abstractions, new mental models, more moving parts. Do not reach for them until you have the problem they solve.

## When to Use Advanced Patterns

```
Do you have this problem?
├── No  → Use standard Rails. Don't add complexity preemptively.
└── Yes → Does a simpler solution exist?
          ├── Yes → Use the simpler solution.
          └── No  → Use the advanced pattern.
```

Standard Rails handles the vast majority of applications. These patterns become relevant at specific inflection points.

## Engines

**Use when:** You have a reusable feature that needs to work across multiple Rails applications, or a large codebase that benefits from hard module boundaries.

**Signals you need it:**
- You are copying code between two Rails apps
- A bounded domain (billing, authentication, CMS) is growing in isolation
- You want to publish an internal gem that multiple teams consume

**Do not use when:** You have a single app and are trying to organize code. Use namespaced controllers, concerns, and service objects instead.

See `engines.md`.

## Internationalization (i18n)

**Use when:** Your application serves users who read different languages, or you need locale-aware formatting for dates, numbers, and currencies.

**Signals you need it:**
- Product requirement to support multiple languages
- User-facing strings that change based on region
- Date/currency formatting that varies by locale

**Do not use when:** You only ever ship in one language and have no plans to change. Adding i18n structure to a monolingual app is overhead with no benefit.

See `i18n.md`.

## Multi-Tenancy

**Use when:** Your application serves multiple distinct customers (tenants), each with their own isolated data.

**Signals you need it:**
- B2B SaaS where each organization's data must be isolated
- Customers must not see each other's records
- You are building a platform product, not an internal tool

**Do not use when:** You are building an application with multiple users who share data (that is standard user management, not multi-tenancy). Do not add tenant isolation if a single organization will use the app.

See `multi-tenancy.md`.

## Files in This Skill

| File | What it covers |
|---|---|
| `engines.md` | Mountable engines, gem extraction, directory structure, migrations |
| `i18n.md` | Locale files, translation helpers, pluralization, date/number formatting |
| `multi-tenancy.md` | Schema-based, row-based, and database-based isolation strategies |
