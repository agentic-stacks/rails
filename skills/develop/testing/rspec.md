# RSpec

RSpec is the most widely used alternative to Minitest in the Rails ecosystem. It offers a DSL focused on readability, a rich matcher library, and flexible shared examples.

---

## Setup

Add to `Gemfile`:

```ruby
group :development, :test do
  gem "rspec-rails", "~> 7.0"
end
```

Install and initialize:

```bash
bundle install
bin/rails generate rspec:install
```

This creates:

```
.rspec                  # default flags: --require spec_helper
spec/
  spec_helper.rb        # RSpec configuration
  rails_helper.rb       # Rails-aware configuration, loads the app
```

Add useful defaults to `.rspec`:

```
--require spec_helper
--format documentation
--color
```

---

## Model Specs

Model specs live in `spec/models/`. Focus on validations, scopes, and methods.

```ruby
# spec/models/article_spec.rb
require "rails_helper"

RSpec.describe Article, type: :model do
  subject(:article) { build(:article) }  # FactoryBot — see factories.md

  describe "validations" do
    it "is valid with all required attributes" do
      expect(article).to be_valid
    end

    it "is invalid without a title" do
      article.title = nil
      expect(article).not_to be_valid
      expect(article.errors[:title]).to include("can't be blank")
    end

    it "is invalid when title exceeds 255 characters" do
      article.title = "a" * 256
      expect(article).not_to be_valid
    end
  end

  describe ".published" do
    let!(:published) { create(:article, status: :published) }
    let!(:draft)     { create(:article, status: :draft) }

    it "includes published articles" do
      expect(Article.published).to include(published)
    end

    it "excludes draft articles" do
      expect(Article.published).not_to include(draft)
    end
  end

  describe "#word_count" do
    it "returns the number of words in the body" do
      article.body = "one two three"
      expect(article.word_count).to eq(3)
    end

    it "returns 0 for an empty body" do
      article.body = ""
      expect(article.word_count).to eq(0)
    end
  end
end
```

### let vs let!

| Helper | When it runs | Use when |
|---|---|---|
| `let(:x) { ... }` | Lazily — first time `x` is called | Default. Avoids unnecessary DB writes. |
| `let!(:x) { ... }` | Eagerly — before each example | The record must exist in the DB even if the test doesn't call `x` directly (e.g., testing a scope that queries all records). |

---

## Request Specs

Request specs replace controller specs. They test the full HTTP stack — router, middleware, controller, model — without a browser.

```ruby
# spec/requests/articles_spec.rb
require "rails_helper"

RSpec.describe "Articles", type: :request do
  let(:user) { create(:user) }
  let(:article) { create(:article, user: user) }

  describe "GET /articles" do
    it "returns HTTP 200" do
      get articles_path
      expect(response).to have_http_status(:ok)
    end

    it "renders the article list" do
      article  # trigger let — create the record
      get articles_path
      expect(response.body).to include(article.title)
    end
  end

  describe "GET /articles/:id" do
    it "returns HTTP 200 for a published article" do
      get article_path(article)
      expect(response).to have_http_status(:ok)
    end
  end

  describe "POST /articles" do
    context "when authenticated" do
      before { sign_in user }  # Devise helper; roll your own otherwise

      it "creates an article and redirects" do
        expect {
          post articles_path, params: { article: { title: "New", body: "Content" } }
        }.to change(Article, :count).by(1)

        expect(response).to redirect_to(article_path(Article.last))
      end

      it "renders the form again on invalid input" do
        post articles_path, params: { article: { title: "", body: "" } }
        expect(response).to have_http_status(:unprocessable_entity)
      end
    end

    context "when not authenticated" do
      it "redirects to login" do
        post articles_path, params: { article: { title: "New", body: "Content" } }
        expect(response).to redirect_to(login_path)
      end
    end
  end

  describe "DELETE /articles/:id" do
    before { sign_in user }

    it "destroys the article" do
      article  # create record
      expect {
        delete article_path(article)
      }.to change(Article, :count).by(-1)
    end
  end
end
```

---

## Matchers

### Built-in RSpec Matchers

```ruby
expect(value).to eq(expected)          # ==
expect(value).to be(expected)          # equal? (object identity)
expect(value).to be_truthy
expect(value).to be_falsy
expect(value).to be_nil
expect(value).to be > 5
expect(value).to be_between(1, 10)
expect(string).to include("substring")
expect(array).to include(element)
expect(array).to match_array([1, 2, 3]) # same elements, any order
expect(array).to contain_exactly(1, 2, 3)
expect(hash).to include(key: value)
expect(object).to be_a(ClassName)
expect(object).to respond_to(:method_name)
expect { code }.to raise_error(ErrorClass)
expect { code }.to raise_error(ErrorClass, "message")
expect { code }.to change(Model, :count).by(1)
expect { code }.to change { Model.count }.from(0).to(1)
expect { code }.not_to change(Model, :count)
```

### shoulda-matchers

Add `shoulda-matchers` for concise model and controller assertions:

```ruby
# Gemfile
group :test do
  gem "shoulda-matchers", "~> 6.0"
end
```

```ruby
# spec/rails_helper.rb
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

```ruby
RSpec.describe Article, type: :model do
  # Associations
  it { is_expected.to belong_to(:user) }
  it { is_expected.to have_many(:comments).dependent(:destroy) }

  # Validations
  it { is_expected.to validate_presence_of(:title) }
  it { is_expected.to validate_length_of(:title).is_at_most(255) }
  it { is_expected.to validate_uniqueness_of(:slug) }
  it { is_expected.to validate_numericality_of(:views).is_greater_than_or_equal_to(0) }

  # Database
  it { is_expected.to have_db_column(:title).of_type(:string) }
  it { is_expected.to have_db_index(:slug).unique(true) }
end
```

---

## Shared Examples

Extract shared behavior to avoid duplication across specs.

### Define

```ruby
# spec/support/shared_examples/requires_authentication.rb
RSpec.shared_examples "requires authentication" do |method, path_helper|
  it "redirects unauthenticated requests to login" do
    send(method, send(path_helper))
    expect(response).to redirect_to(login_path)
  end
end
```

### Use

```ruby
RSpec.describe "Articles", type: :request do
  include_examples "requires authentication", :get, :articles_path
  include_examples "requires authentication", :post, :articles_path
end
```

### Shared Contexts

```ruby
# spec/support/shared_contexts/authenticated_user.rb
RSpec.shared_context "authenticated user" do
  let(:user) { create(:user) }
  before { sign_in user }
end

# Use in specs
RSpec.describe "Articles", type: :request do
  include_context "authenticated user"

  it "allows creating articles" do
    # user is already signed in
  end
end
```

---

## Mocking and Stubbing

```ruby
# Stub a method on an instance
allow(article).to receive(:word_count).and_return(42)

# Stub a class method
allow(Article).to receive(:published).and_return([article])

# Expect a method to be called (verification double)
expect(ArticleMailer).to receive(:published_notification).with(article).and_call_original

# Stub an external service
allow(SlackNotifier).to receive(:notify).and_return(true)

# Raise an error from a stub
allow(PaymentGateway).to receive(:charge).and_raise(PaymentGateway::NetworkError)

# Partial doubles: only stub specific methods
allow_any_instance_of(Article).to receive(:slug).and_return("stubbed-slug")
# Prefer instance-level stubs over allow_any_instance_of when possible
```

Use `instance_double`, `class_double`, and `object_double` for verified doubles that enforce the real interface:

```ruby
mailer = instance_double(ArticleMailer)
allow(ArticleMailer).to receive(:with).and_return(mailer)
allow(mailer).to receive(:deliver_later)
```

---

## Running RSpec

```bash
# Run all specs
bundle exec rspec

# Run a directory
bundle exec rspec spec/models/

# Run a specific file
bundle exec rspec spec/models/article_spec.rb

# Run a specific line
bundle exec rspec spec/models/article_spec.rb:15

# Run only examples matching a pattern
bundle exec rspec --example "word count"

# Run with seed for reproducible ordering
bundle exec rspec --seed 12345

# Show slowest examples
bundle exec rspec --profile 10
```

Tag and filter:

```ruby
it "hits the payment gateway", :slow do ... end
it "integration test", :integration do ... end
```

```bash
bundle exec rspec --tag slow
bundle exec rspec --tag ~slow   # exclude slow
```
