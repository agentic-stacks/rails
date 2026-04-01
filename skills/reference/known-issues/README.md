# Known Issues

Rails-version-specific bugs, gotchas, and their workarounds.

## Purpose

This directory documents known issues encountered when working with specific Rails versions. Entries cover framework bugs, gem incompatibilities, tooling gaps, and configuration pitfalls that are not immediately obvious from the official documentation.

Use these files when:

- A newly generated app behaves unexpectedly
- An upgrade introduces a regression
- A community workaround exists before an upstream fix lands
- Symptoms are cryptic and the root cause is a known framework issue

## Naming Convention

Files are named `rails-{major}.md`:

| File | Covers |
|---|---|
| `rails-7.md` | Rails 7.0, 7.1, 7.2 |
| `rails-8.md` | Rails 8.0 |

When a new major version ships, add `rails-{major}.md`. Do not collapse versions — keeping them separate makes it easy to delete an entire file once support for that major ends.

## Entry Format

Each issue uses this structure:

```
### [Short Title]

**Symptom:** What you observe — error message, wrong output, silent failure.

**Cause:** Why it happens — framework bug, config default, gem conflict, etc.

**Affected versions:** e.g. 7.0.0–7.0.6, fixed in 7.0.7

**Status:** One of: `open`, `fixed in X.Y.Z`, `workaround only`, `upstream`

**Workaround:**
Steps or code to resolve the issue until a fix lands.
```

## Status Values

| Status | Meaning |
|---|---|
| `open` | No fix or workaround yet confirmed |
| `fixed in X.Y.Z` | Resolved in the listed patch release |
| `workaround only` | No upstream fix expected; use the workaround |
| `upstream` | Issue is in a dependency, not Rails itself; link to upstream tracker |

## Contributing an Entry

1. Verify the issue is reproducible and not caused by local configuration.
2. Search the Rails issue tracker and changelogs to confirm status.
3. Add the entry under the correct major version file in version-ascending order.
4. Include a link to the upstream issue or PR when available.
5. Update `status` once a fix ships and note the resolved version.
