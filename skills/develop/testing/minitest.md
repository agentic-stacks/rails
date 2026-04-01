# Minitest

Minitest is Rails' default test framework. It ships with Rails — no gems required. Tests live in `test/`, organized by layer.

```
test/
  models/            # Unit tests for ActiveRecord models
  controllers/       # Controller unit tests (rarely used — prefer integration)
  integration/       # Multi-step request/response tests
  system/            # Browser tests (covered in system-tests.md)
  helpers/
  mailers/
  jobs/
  fixtures/          # YAML test data
  test_helper.rb     # Global test configuration
  application_system_test_case.rb
```

---

## Model Tests

Model tests live in `test/models/`. They test validations, scopes, and business logic without touching the HTTP stack.

```ruby
# test/models/article_test.rb
require "test_helper"

class ArticleTest < ActiveSupport::TestCase
  # Fixtures are available by name
  test "valid with title and body" do
    article = Article.new(title: "Hello", body: "World")
    assert article.valid?
  end

  test "invalid without title" do
    article = Article.new(body: "World")
    assert_not article.valid?
    assert_includes article.errors[:title], "can't be blank"
  end

  test "invalid without body" do
    article = Article.new(title: "Hello")
    assert_not article.valid?
  end

  test "published scope returns only published articles" do
    published = articles(:published)
    draft = articles(:draft)

    results = Article.published
    assert_includes results, published
    assert_not_includes results, draft
  end

  test "word_count returns number of words in body" do
    article = Article.new(body: "one two three")
    assert_equal 3, article.word_count
  end
end
```

---

## Core Assertions

| Assertion | Passes when |
|---|---|
| `assert expr` | `expr` is truthy |
| `assert_not expr` | `expr` is falsy |
| `assert_equal expected, actual` | `expected == actual` |
| `assert_not_equal a, b` | `a != b` |
| `assert_nil obj` | `obj.nil?` |
| `assert_not_nil obj` | `!obj.nil?` |
| `assert_empty collection` | `collection.empty?` |
| `assert_includes collection, obj` | `collection.include?(obj)` |
| `assert_match pattern, string` | `string =~ pattern` |
| `assert_raises(ErrorClass) { block }` | block raises the named error |
| `assert_difference "Model.count", 1 { block }` | count increases by 1 |
| `assert_no_difference "Model.count" { block }` | count unchanged |
| `assert_changes -> { expr }, from: x, to: y { block }` | expr changes as specified |

```ruby
test "create increments article count" do
  assert_difference "Article.count", 1 do
    Article.create!(title: "New", body: "Content")
  end
end

test "raises RecordNotFound for missing id" do
  assert_raises(ActiveRecord::RecordNotFound) do
    Article.find(-1)
  end
end

test "publishing changes status" do
  article = articles(:draft)
  assert_changes -> { article.status }, from: "draft", to: "published" do
    article.publish!
  end
end
```

---

## Setup and Teardown

```ruby
class ArticleTest < ActiveSupport::TestCase
  setup do
    @user = users(:admin)
    @article = Article.new(title: "Test", body: "Body", user: @user)
  end

  teardown do
    # Called after every test — rarely needed with transactional tests
    # Use for cleaning up files, external stubs, etc.
  end

  test "valid article" do
    assert @article.valid?
  end

  test "belongs to user" do
    assert_equal @user, @article.user
  end
end
```

Use `setup` over instance variable initialization in each test. Use `teardown` only when you create side effects that transactions do not clean up (files, external service state, class-level stubs).

---

## Integration Tests

Integration tests send real HTTP requests through the full Rails stack — router, middleware, controller, model — without a browser. Use these to test controller behavior.

> Prefer integration tests over controller tests. Integration tests exercise the full stack and are less brittle.

```ruby
# test/integration/articles_test.rb
require "test_helper"

class ArticlesTest < ActionDispatch::IntegrationTest
  setup do
    @user = users(:admin)
  end

  test "GET /articles returns 200" do
    get articles_url
    assert_response :success
  end

  test "GET /articles/:id shows article" do
    article = articles(:published)
    get article_url(article)
    assert_response :success
    assert_select "h1", article.title
  end

  test "POST /articles creates article when authenticated" do
    sign_in @user  # helper defined in test_helper.rb

    assert_difference "Article.count", 1 do
      post articles_url, params: {
        article: { title: "New Article", body: "Some content" }
      }
    end

    assert_redirected_to article_url(Article.last)
    follow_redirect!
    assert_response :success
  end

  test "POST /articles rejects invalid article" do
    sign_in @user

    assert_no_difference "Article.count" do
      post articles_url, params: {
        article: { title: "", body: "" }
      }
    end

    assert_response :unprocessable_entity
  end

  test "DELETE /articles/:id destroys article" do
    sign_in @user
    article = articles(:published)

    assert_difference "Article.count", -1 do
      delete article_url(article)
    end

    assert_redirected_to articles_url
  end
end
```

### Common Response Assertions

```ruby
assert_response :success          # 200
assert_response :redirect         # 3xx
assert_response :not_found        # 404
assert_response :unprocessable_entity  # 422
assert_response 201               # exact status code

assert_redirected_to articles_url
assert_redirected_to article_url(@article)

follow_redirect!                  # follow a redirect and assert its response
assert_response :success
```

---

## Fixtures

Fixtures are YAML files in `test/fixtures/`. They define test records loaded into the database before each test run.

```yaml
# test/fixtures/articles.yml
published:
  title: Published Article
  body: This article is published.
  status: published
  user: admin          # references users fixture by label
  created_at: <%= 2.days.ago %>
  updated_at: <%= 2.days.ago %>

draft:
  title: Draft Article
  body: This article is a draft.
  status: draft
  user: admin
```

```yaml
# test/fixtures/users.yml
admin:
  email: admin@example.com
  name: Admin User
  role: admin
  password_digest: <%= BCrypt::Password.create("password") %>
```

### Using Fixtures in Tests

```ruby
# Fetch a fixture by label — returns an ActiveRecord instance
article = articles(:published)
user = users(:admin)

# Pass multiple labels to get an array
first, second = articles(:published, :draft)
```

### ERB in Fixtures

Fixtures support ERB for dynamic values:

```yaml
# test/fixtures/articles.yml
recent:
  title: Recent Article
  body: Written recently.
  created_at: <%= 1.hour.ago %>
  published_at: <%= Time.zone.now %>

<%# Generate many fixtures with a loop %>
<% 10.times do |i| %>
article_<%= i %>:
  title: Article <%= i %>
  body: Body <%= i %>
<% end %>
```

### Fixture Caveats

- Fixtures bypass model validations and callbacks — they insert directly via SQL
- All fixtures are loaded for all tests regardless of which test runs
- IDs are deterministic (derived from label) so associations work across fixture files
- Use `articles(:label)` not `Article.find(1)` — IDs vary across environments

---

## Parallel Testing

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)  # use all CPU cores
  # parallelize(workers: 4)                    # explicit count
end
```

Parallel tests run in forked processes. Each process gets its own database (`test_db_0`, `test_db_1`, etc.) created automatically.

```bash
# Run tests with 4 parallel workers
PARALLEL_WORKERS=4 bin/rails test

# Run system tests in parallel (Rails 7.1+)
PARALLEL_WORKERS=2 bin/rails test:system
```

Set up and tear down each worker database if needed:

```ruby
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)

  parallelize_setup do |worker|
    # Seed worker-specific data after database is set up
  end

  parallelize_teardown do |worker|
    # Clean up after the worker finishes
  end
end
```
