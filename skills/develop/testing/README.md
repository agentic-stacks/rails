# Testing

Testing is not optional. Untested Rails code is technical debt with a fuse. This skill covers how to think about tests, what to test, and how to run them.

---

## The Test Pyramid

Write tests at three levels. The distribution matters.

```
        /\
       /  \        System tests (few)
      / UI \       Full browser, Capybara + Chrome
     /------\
    /        \     Integration / Request tests (moderate)
   / Requests \    HTTP in, HTTP out — no browser
  /------------\
 /              \  Unit tests (many)
/   Models etc.  \ Models, jobs, mailers, helpers, POROs
/------------------\
```

| Layer | What it tests | Speed | Fragility | Count |
|---|---|---|---|---|
| Unit | Models, service objects, POROs | Very fast | Low | Most |
| Integration | Controller actions, request/response cycle | Fast | Moderate | Some |
| System | Full user flows in a browser | Slow | High | Few |

Run units constantly. Run integration on every commit. Run system tests in CI and before deploy.

---

## What to Test

### Always test

- **Validations** — every `validates` rule has at least one passing and one failing case
- **Business logic** — any method that computes, transforms, or decides
- **Scopes** — confirm the SQL returns the right records
- **State transitions** — if a model has a status field, test each valid transition
- **Controller responses** — status codes, redirects, flash messages
- **Authorization** — confirm unpermitted users cannot perform privileged actions
- **Background jobs** — that they enqueue with the right arguments and perform correctly
- **Edge cases** — nil, empty string, boundary values, concurrent writes

### Do not test

- **Framework code** — `ActiveRecord::Base`, `ActionController::Base`, and Rails internals are already tested by the Rails core team
- **Getters/setters** — `user.name` returning what you set is not a test
- **Database schema** — column existence is a migration concern, not a test concern
- **Third-party gems** — trust the gem's own test suite; test only your wrapper
- **Incidental rendering** — do not write tests that assert exact HTML structure unless you own that structure

---

## Running Tests

### Minitest (Rails default)

```bash
# Run all tests
bin/rails test

# Run a single file
bin/rails test test/models/article_test.rb

# Run a specific line
bin/rails test test/models/article_test.rb:42

# Run system tests separately (they are excluded from `bin/rails test`)
bin/rails test:system

# Run everything including system tests
bin/rails test:all
```

### RSpec

```bash
# Run all specs
bundle exec rspec

# Run a file
bundle exec rspec spec/models/article_spec.rb

# Run a specific line
bundle exec rspec spec/models/article_spec.rb:15

# Run with documentation format
bundle exec rspec --format documentation
```

---

## CI Setup

Prepare the test database before running any tests:

```bash
# Create and load schema (idempotent in CI)
RAILS_ENV=test bin/rails db:create db:schema:load

# Or if you need to re-apply all migrations
RAILS_ENV=test bin/rails db:drop db:create db:schema:load
```

Use `db:schema:load` over `db:migrate` in CI — it is faster and avoids cumulative migration bugs.

### Parallel Tests

Rails has built-in parallel test support. Enable it in `test/test_helper.rb`:

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)

  # When using fixtures with parallel tests, prepare each worker's database:
  parallelize_setup do |worker|
    ActiveRecord::TestFixtures.create_fixtures(
      ActiveRecord::Base.connection,
      Dir["#{Rails.root}/test/fixtures/*.yml"].map { |f| File.basename(f, ".yml") }
    )
  end
end
```

Run with explicit worker count:

```bash
PARALLEL_WORKERS=4 bin/rails test
```

For RSpec, use the `parallel_tests` gem:

```bash
bundle exec parallel_rspec spec/
```

---

## Test Database State

Each test runs in a transaction that is rolled back after the test completes. This keeps tests isolated without truncating tables between each run.

```ruby
# test/test_helper.rb — this is the default
class ActiveSupport::TestCase
  use_transactional_tests = true  # default: true
end
```

System tests use a real browser that runs outside the test process, so they cannot share the transaction. They use database truncation or deletion strategies instead. Configure this in `test/application_system_test_case.rb` (Minitest) or `spec/support/database_cleaner.rb` (RSpec).

---

## Files in This Skill

| File | What it covers |
|---|---|
| `minitest.md` | Rails default test framework: models, integration, fixtures, assertions |
| `rspec.md` | RSpec setup, request specs, matchers, shared examples, mocking |
| `system-tests.md` | Capybara, browser drivers, debugging techniques |
| `factories.md` | FactoryBot vs fixtures, traits, sequences, associations |

---

## Next Steps

- Write model and integration tests: `minitest.md`
- Adopt RSpec: `rspec.md`
- Test user flows in a browser: `system-tests.md`
- Generate test data cleanly: `factories.md`
