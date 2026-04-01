# Decision Guides

Structured comparisons to help choose between Rails tooling options.

## Purpose

Decision guides exist for choices where multiple good options exist and the right answer depends on your project's constraints. Each guide provides a comparison table, trade-off analysis, and a recommendation by use case.

Use a decision guide when:

- Starting a new app and evaluating defaults vs. alternatives
- Upgrading and needing to decide whether to migrate to a new default
- Onboarding a team member who asks "why did we pick X over Y?"

## Guides

| Guide | Question Answered |
|---|---|
| [asset-pipeline.md](asset-pipeline.md) | Propshaft + importmap vs. Propshaft + esbuild vs. Sprockets |
| [background-jobs.md](background-jobs.md) | Solid Queue vs. Sidekiq vs. GoodJob |
| [authentication.md](authentication.md) | Rails 8 generator vs. `has_secure_password` vs. Devise |
| [testing-framework.md](testing-framework.md) | Minitest vs. RSpec |

## How to Use a Guide

1. Read the comparison table to understand what each option provides.
2. Find your use case in the "Recommendations" section.
3. Check any referenced known-issues files for version-specific gotchas.
4. Record the decision and rationale in your project's ADR (Architecture Decision Record) or equivalent.

## When Not to Use a Guide

- If your organisation has a mandate (e.g. "we use Sidekiq everywhere"), follow the mandate.
- If the app already uses one option and migration cost is high, treat switching as a separate decision.
- If you need a feature that only one option provides, the decision is already made — skip to the implementation docs.

## Adding a New Guide

Name the file `{topic}.md` and follow this structure:

1. One-sentence framing of the decision
2. Comparison table with consistent columns
3. Detail section for each option (strengths, weaknesses, when to avoid)
4. Recommendations by use case
5. Migration path (if applicable)
