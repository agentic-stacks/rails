# Factories and Fixtures

Every test needs data. Rails provides fixtures built-in. FactoryBot is the most popular alternative. Know both — choose deliberately.

---

## Fixtures: Rails Default

Fixtures are YAML files in `test/fixtures/`. They load directly into the database before the test suite runs.

```yaml
# test/fixtures/articles.yml
published_article:
  title: A Published Article
  body: The full body of the article.
  status: published
  user: admin_user

draft_article:
  title: A Draft
  body: Not ready yet.
  status: draft
  user: admin_user
```

```yaml
# test/fixtures/users.yml
admin_user:
  email: admin@example.com
  name: Admin
  role: admin
  password_digest: <%= BCrypt::Password.create("password") %>
```

Use fixture records in tests by label:

```ruby
user    = users(:admin_user)
article = articles(:published_article)
```

### Fixture Pros

- Zero gem dependencies — ships with Rails
- Fast — inserted in bulk once per test run, not before every test
- Simple YAML format
- Deterministic IDs derived from label names (safe for cross-fixture associations)

### Fixture Cons

- Shared global state — every test sees every fixture
- Bypass validations and callbacks — records may be in a state your app would never create
- Hard to maintain as the schema grows — YAML becomes unwieldy for complex data
- No dynamic variation — generating ten similar-but-different records requires ERB loops
- Poor fit for RSpec, which does not integrate fixtures natively (though it is possible)

### When to Use Fixtures

Use fixtures in:
- Small Rails apps with simple data models
- Apps that will stay on Minitest
- Apps where the fixture set is stable and small

---

## FactoryBot

FactoryBot generates model instances in Ruby. Each factory defines a valid object by default; tests customize only what they need.

```ruby
# Gemfile
group :development, :test do
  gem "factory_bot_rails"
end
```

Factories live in `spec/factories/` (RSpec) or `test/factories/` (Minitest):

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    name { "Test User" }
    role { :member }
    password { "password" }
  end
end
```

```ruby
# spec/factories/articles.rb
FactoryBot.define do
  factory :article do
    sequence(:title) { |n| "Article #{n}" }
    body { "Default article body with enough content." }
    status { :draft }
    association :user

    trait :published do
      status { :published }
      published_at { 1.day.ago }
    end

    trait :draft do
      status { :draft }
      published_at { nil }
    end

    trait :with_comments do
      after(:create) do |article|
        create_list(:comment, 3, article: article)
      end
    end
  end
end
```

---

## Build Strategies

| Strategy | What it does | Hits the DB? |
|---|---|---|
| `create(:article)` | Saves to the database | Yes |
| `build(:article)` | Builds an unsaved instance | No |
| `build_stubbed(:article)` | Builds a fake persisted instance (has an ID, stubs `persisted?`) | No |
| `attributes_for(:article)` | Returns a plain Hash of attributes | No |

```ruby
# Saved instance — use in integration/system tests
article = create(:article)

# Unsaved instance — use in model unit tests where you don't need DB
article = build(:article)

# Fast fake persisted instance — use for service object / decorator tests
article = build_stubbed(:article)

# Hash of attributes — use for submitting params in request tests
params = attributes_for(:article)
post articles_path, params: { article: params }
```

Default to `build` or `build_stubbed` in model specs — it is faster. Use `create` only when the test requires a real persisted record (queries, associations, callbacks on commit).

---

## Traits

Traits define named variations of a factory. Compose them to produce specific states.

```ruby
FactoryBot.define do
  factory :article do
    title { "Default Title" }
    body  { "Default body." }
    status { :draft }
    association :user

    trait :published do
      status { :published }
      published_at { 1.day.ago }
    end

    trait :long do
      body { "word " * 500 }
    end

    trait :with_cover_image do
      after(:build) do |article|
        article.cover_image.attach(
          io: File.open(Rails.root.join("spec/fixtures/files/cover.jpg")),
          filename: "cover.jpg",
          content_type: "image/jpeg"
        )
      end
    end
  end
end
```

```ruby
# Use single trait
create(:article, :published)

# Compose multiple traits
create(:article, :published, :with_cover_image)

# Override specific attributes inline
create(:article, :published, title: "Custom Title")
```

---

## Sequences

Sequences generate unique values. Use them for fields with uniqueness constraints.

```ruby
FactoryBot.define do
  sequence(:email) { |n| "user#{n}@example.com" }
  sequence(:slug)  { |n| "article-slug-#{n}" }

  factory :user do
    email  # uses the global :email sequence
    sequence(:username) { |n| "user#{n}" }
  end
end
```

---

## Transient Attributes

Transient attributes are not persisted — they control factory behavior without becoming model attributes.

```ruby
FactoryBot.define do
  factory :article do
    title { "Default" }

    transient do
      comment_count { 0 }
    end

    after(:create) do |article, evaluator|
      create_list(:comment, evaluator.comment_count, article: article)
    end
  end
end
```

```ruby
article = create(:article, comment_count: 5)
article.comments.count  # => 5
```

---

## Associations

FactoryBot creates associated records automatically when you use `association`:

```ruby
FactoryBot.define do
  factory :article do
    association :user       # creates a user using the :user factory
    title { "My Article" }
    body  { "Content." }
  end

  factory :comment do
    association :article    # creates an article
    association :user       # creates a user
    body { "A comment." }
  end
end
```

Share a parent record across associated objects to avoid orphaned records and unnecessary DB writes:

```ruby
user    = create(:user)
article = create(:article, user: user)
comment = create(:comment, article: article, user: user)
```

---

## Linting Factories

Lint all factories to catch broken definitions before they cause cryptic test failures:

```ruby
# spec/factories_spec.rb  OR  in RSpec config
RSpec.describe "FactoryBot factories" do
  it "are all valid" do
    FactoryBot.lint
  end
end
```

Or with traits:

```ruby
FactoryBot.lint traits: true  # lints every factory + every trait
```

Add this to your CI pipeline. A factory that cannot produce a valid object is a broken factory.

---

## Minitest Integration

FactoryBot works with Minitest. Include the syntax helpers:

```ruby
# test/test_helper.rb
require "factory_bot_rails"

class ActiveSupport::TestCase
  include FactoryBot::Syntax::Methods
end
```

Then use `create`, `build`, `build_stubbed` without the `FactoryBot.` prefix:

```ruby
class ArticleTest < ActiveSupport::TestCase
  test "publish!" do
    article = build(:article, :draft)
    article.publish!
    assert_equal "published", article.status
  end
end
```

---

## Recommendation

| Situation | Recommendation |
|---|---|
| Simple app, Minitest, few models | Fixtures — zero overhead, fast, built-in |
| Complex domain, many model states | FactoryBot — traits compose cleanly |
| RSpec project | FactoryBot — better native integration |
| Performance-sensitive test suite | Mix: `build_stubbed` for unit tests, fixtures for shared reference data |

Do not use both simultaneously without a plan. Pick one approach per layer. If you add FactoryBot, disable automatic fixture loading for those tests to avoid loading data you don't use.
