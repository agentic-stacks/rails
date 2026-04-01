# Testing Framework Decision Guide

Choose between Minitest and RSpec for your Rails application test suite.

---

## Comparison Table

| Criterion | Minitest | RSpec |
|---|---|---|
| **Ships with Rails** | Yes (default) | No (gem required) |
| **Syntax style** | `assert_*` / `test "..."` | `describe` / `it` / `expect` |
| **Learning curve** | Low | Medium |
| **Documentation** | Rails guides, Ruby stdlib | Dedicated rspec.info + community |
| **Community usage** | Common in open source | Dominant in industry (as of 2024) |
| **Shared examples / behaviours** | Via modules | Native (`shared_examples_for`) |
| **Mocking / stubbing** | `Minitest::Mock` (basic) or `mocha` gem | `allow`, `expect`, `double` (built-in) |
| **Matchers** | Limited (assert style) | Rich (hundreds of built-in matchers) |
| **Custom matchers** | Manual | Easy via `RSpec::Matchers.define` |
| **Spec-style output** | Via `minitest-reporters` gem | Native (`--format documentation`) |
| **Test speed** | Fast (minimal overhead) | Slightly slower (larger framework) |
| **Parallelism** | Native (`parallelize`) | Via `parallel_tests` gem |
| **FactoryBot integration** | Works | Works |
| **Rails system tests** | Yes (Capybara, Selenium) | Yes (via `rspec-rails`) |
| **Let / subject / before** | Manual setup (`setup` method) | Native |
| **Pending tests** | `skip` | `pending`, `xit` |
| **Focus / filter** | `-n /pattern/` flag | `fit`, `fdescribe`, `--tag` |
| **Configuration** | Minimal (`test_helper.rb`) | `.rspec`, `spec/spec_helper.rb`, `spec/rails_helper.rb` |
| **Ecosystem depth** | Moderate | Very deep (`rspec-*` gems) |

---

## Option Details

### Minitest

Minitest is bundled with Ruby and integrated into Rails. All Rails generators produce Minitest files by default. It uses assertion-based syntax (`assert_equal`, `assert_nil`, etc.) and inherits from `ActiveSupport::TestCase`.

```ruby
# test/models/user_test.rb
class UserTest < ActiveSupport::TestCase
  test "requires email" do
    user = User.new(email: nil)
    assert_not user.valid?
    assert_includes user.errors[:email], "can't be blank"
  end
end
```

**Strengths:**
- No extra gems to install or configure
- All Rails documentation and guides use Minitest
- Extremely fast — minimal boot overhead
- `assert_*` style is easy to explain to developers new to testing
- Rails generators generate test files automatically — nothing extra to set up
- Native parallel test support (`parallelize(workers: :number_of_processors)`)

**Weaknesses:**
- `assert_equal expected, actual` argument order is easy to get backwards (error messages will be confusing)
- Shared behaviour requires Ruby modules — verbose compared to RSpec shared examples
- Built-in mocking (`Minitest::Mock`) is limited; production use often requires `mocha` or `rr`
- Less expressive output by default (pass/fail dots); readable output requires `minitest-reporters`
- No `let`, `subject`, or `before` — setup methods can become large

---

### RSpec

RSpec is the most widely used Ruby testing framework in professional Rails development. It uses a `describe`/`it`/`expect` DSL designed to read as a specification.

```ruby
# spec/models/user_spec.rb
RSpec.describe User do
  describe "validations" do
    it "requires email" do
      user = User.new(email: nil)
      expect(user).not_to be_valid
      expect(user.errors[:email]).to include("can't be blank")
    end
  end
end
```

Install:
```ruby
# Gemfile
group :development, :test do
  gem "rspec-rails"
end
```
```bash
bin/rails generate rspec:install
```

**Strengths:**
- Expressive, human-readable test descriptions
- Rich built-in matchers (`include`, `change { }.by`, `raise_error`, `have_attributes`, etc.)
- `let`, `let!`, `subject`, `before`, `after`, `around` hooks keep setup DRY
- Shared examples and shared contexts reduce duplication across similar specs
- Excellent mocking/stubbing DSL built in — `instance_double`, `allow`, `expect(...).to receive`
- `--format documentation` produces clear spec-style output
- Very large ecosystem: `rspec-matchers`, `shoulda-matchers`, `webmock`, `vcr`, etc.
- Focus mode (`fit`, `fdescribe`) for running a subset without changing config

**Weaknesses:**
- Requires `rspec-rails` gem and separate install step
- Rails generators do not produce RSpec files without a generator override or generator config
- Larger configuration surface (`spec_helper.rb`, `rails_helper.rb`, `.rspec`, shared contexts)
- Boot time is slightly higher than Minitest
- `let` is lazy — `let!` forces eager evaluation; the difference trips up newcomers
- Shared examples are powerful but can make test origins hard to trace

---

## Recommendations by Use Case

| Use Case | Recommended | Rationale |
|---|---|---|
| New Rails app, small team, or solo project | Minitest | Default Rails integration, lower setup, sufficient for most apps |
| New app where the team has strong RSpec experience | RSpec | Familiarity reduces ramp-up; RSpec's expressiveness pays off at scale |
| API project with complex behavioural requirements | RSpec | Shared examples and rich matchers shine for API contracts |
| Open source library or engine | Minitest | Consistent with Rails core and most gem conventions |
| Large app with many similar model specs | RSpec | Shared examples and `shoulda-matchers` reduce repetition significantly |
| Existing Rails app already using one | Keep the existing framework | Consistency matters more than which one you use |
| Teaching Rails testing to beginners | Minitest | Smaller surface area; same framework as Rails docs |

---

## Coexistence and Migration

### Can you use both?

Yes — `rspec-rails` and Minitest can coexist in the same project. Some teams run Minitest for system tests (where Rails generators write them) and RSpec for unit and integration specs. This is not recommended — pick one.

### Migrating from Minitest to RSpec

There is no automated migration tool. The practical approach:

1. Install `rspec-rails` and run `rails generate rspec:install`.
2. Leave existing Minitest tests in place — they still run with `bin/rails test`.
3. Write all new tests in RSpec under `spec/`.
4. Gradually convert existing tests file-by-file when touching them.
5. Once the `test/` directory is empty (or nearly so), remove `minitest` configuration.

### Migrating from RSpec to Minitest

Less common. The same gradual approach applies: write new tests in Minitest under `test/`, convert old specs when touching those files.

---

## Ecosystem Add-ons

### Works with either framework

| Gem | Purpose |
|---|---|
| `factory_bot_rails` | Test data factories |
| `faker` | Fake data generation |
| `webmock` | Stub HTTP requests |
| `vcr` | Record and replay HTTP interactions |
| `capybara` | Browser integration testing |
| `selenium-webdriver` | Browser driver for system tests |
| `cuprite` | Chrome-native driver (no Selenium) |
| `database_cleaner-active_record` | Control DB state between tests |

### Minitest-specific

| Gem | Purpose |
|---|---|
| `minitest-reporters` | Coloured, spec-format output |
| `mocha` | More expressive mocking than `Minitest::Mock` |
| `shoulda-context` | RSpec-like `context` blocks in Minitest |
| `shoulda-matchers` | One-liner model and controller matchers |

### RSpec-specific

| Gem | Purpose |
|---|---|
| `rspec-rails` | Rails integration (required) |
| `shoulda-matchers` | One-liner model and controller matchers |
| `rspec-sidekiq` | Job matchers for Sidekiq |
| `rspec-graphql_matchers` | GraphQL schema matchers |
| `pundit-matchers` | Policy matchers for Pundit |
| `parallel_tests` | Parallel spec execution |
