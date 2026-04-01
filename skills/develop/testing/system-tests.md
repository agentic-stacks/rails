# System Tests

System tests exercise your application through a real browser. They catch integration failures — broken JavaScript, missing Turbo responses, incorrect redirects — that request tests cannot see.

Use system tests sparingly. They are slow and sensitive to timing. Cover the critical user paths: sign in, core CRUD flows, checkout. Let unit and integration tests handle the rest.

---

## What System Tests Are

A system test starts a real browser, visits your app, and interacts with it like a user: clicking buttons, filling forms, following links. Capybara drives the browser. The browser driver (Selenium, Playwright) translates Capybara commands into browser actions.

```
Your test
  └─► Capybara DSL (visit, fill_in, click_on)
        └─► Driver (Selenium / Playwright)
              └─► Browser (Chrome / Firefox / WebKit)
                    └─► Your Rails app (Puma test server)
```

---

## Setup for Minitest

Rails ships with system test support. The default driver is `selenium_chrome_headless`.

```ruby
# test/application_system_test_case.rb
require "test_helper"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :headless_chrome, screen_size: [1400, 900]
end
```

Generate a system test:

```bash
bin/rails generate system_test articles
# creates test/system/articles_test.rb
```

```ruby
# test/system/articles_test.rb
require "application_system_test_case"

class ArticlesTest < ApplicationSystemTestCase
  test "creating an article" do
    visit new_article_url

    fill_in "Title", with: "My First Article"
    fill_in "Body", with: "This is the content."
    click_on "Create Article"

    assert_text "Article was successfully created"
    assert_text "My First Article"
  end

  test "listing articles" do
    article = articles(:published)
    visit articles_url
    assert_text article.title
  end
end
```

---

## Setup for RSpec

Install the required gems:

```ruby
# Gemfile
group :test do
  gem "capybara"
  gem "selenium-webdriver"
end
```

Configure Capybara in `spec/rails_helper.rb` or a support file:

```ruby
# spec/support/capybara.rb
require "capybara/rspec"

RSpec.configure do |config|
  config.before(:each, type: :system) do
    driven_by :selenium, using: :headless_chrome, screen_size: [1400, 900]
  end
end
```

Write system specs with `type: :system`:

```ruby
# spec/system/articles_spec.rb
require "rails_helper"

RSpec.describe "Articles", type: :system do
  let(:user) { create(:user) }

  before { sign_in user }

  it "creates an article" do
    visit new_article_path

    fill_in "Title", with: "My First Article"
    fill_in "Body", with: "This is the content."
    click_on "Create Article"

    expect(page).to have_text("Article was successfully created")
    expect(page).to have_text("My First Article")
  end

  it "edits an article" do
    article = create(:article, user: user)
    visit edit_article_path(article)

    fill_in "Title", with: "Updated Title"
    click_on "Update Article"

    expect(page).to have_text("Article was successfully updated")
    expect(page).to have_text("Updated Title")
  end
end
```

---

## Capybara DSL

### Navigation

```ruby
visit articles_path
visit "https://example.com/articles"

# Current page
current_path   # => "/articles"
current_url    # => "http://127.0.0.1:3000/articles"
```

### Finders

```ruby
find("h1")                          # raises if not found
find("h1", text: "My Article")
find(:css, ".article-title")
find(:xpath, "//h1")
find_by_id("article-form")

all("li.article")                   # returns array-like NodeSet
first("li.article")

# Scope interactions to a section
within("#article-form") do
  fill_in "Title", with: "Scoped"
end

within_table("articles") do
  expect(page).to have_text("My Article")
end
```

### Interactions

```ruby
click_on "Create Article"           # matches links and buttons by text
click_link "View"
click_button "Submit"

fill_in "Title", with: "Hello"
fill_in "Email", with: "user@example.com"

check "Accept terms"
uncheck "Subscribe to newsletter"

choose "Female"                     # radio button

select "Published", from: "Status"
unselect "Draft", from: "Status"    # for multi-selects

attach_file "Avatar", "/path/to/file.jpg"
```

### Assertions (Minitest)

```ruby
assert_text "Article was created"
assert_text "Welcome", wait: 5        # wait up to 5 seconds
assert_no_text "Error"
assert_selector "h1", text: "Articles"
assert_selector "li", count: 3
assert_no_selector ".error-message"
assert_current_path articles_path
assert_link "New Article"
assert_button "Submit"
assert_field "Title", with: "My Article"
```

### Assertions (RSpec)

```ruby
expect(page).to have_text("Article was created")
expect(page).to have_text("Welcome", wait: 5)
expect(page).not_to have_text("Error")
expect(page).to have_selector("h1", text: "Articles")
expect(page).to have_selector("li", count: 3)
expect(page).to have_current_path(articles_path)
expect(page).to have_link("New Article")
expect(page).to have_button("Submit")
expect(page).to have_field("Title", with: "My Article")
```

---

## Drivers

### selenium_chrome_headless (default)

Runs Chrome in headless mode. Fast enough for CI. Requires Chrome and chromedriver installed.

```ruby
driven_by :selenium, using: :headless_chrome, screen_size: [1400, 900]
```

### selenium_chrome (headed)

Useful for debugging locally — opens a visible Chrome window.

```ruby
driven_by :selenium, using: :chrome, screen_size: [1400, 900]
```

### rack_test

No browser. Fastest driver. Cannot run JavaScript. Use only for simple HTML forms.

```ruby
driven_by :rack_test
```

### Playwright (cuprite or playwright-ruby)

An alternative to Selenium that does not require chromedriver. Uses Chrome DevTools Protocol directly.

```ruby
# Gemfile
gem "cuprite"  # CDP-based alternative to Selenium
```

```ruby
# spec/support/capybara.rb
require "capybara/cuprite"

RSpec.configure do |config|
  config.before(:each, type: :system) do
    driven_by :cuprite, using: :chrome, screen_size: [1400, 900]
  end
end
```

Cuprite is faster than Selenium and does not depend on a matching chromedriver version.

---

## Debugging System Tests

### Take a Screenshot

```ruby
# Minitest
take_screenshot  # saves to tmp/screenshots/

# RSpec / Capybara
save_screenshot("tmp/screenshots/debug.png")

# Automatic screenshots on failure (Minitest)
# Already enabled in ApplicationSystemTestCase:
# self.use_screenshots = true  (default)
```

### Open the Page

```ruby
save_and_open_page   # opens the HTML in your browser
```

### Print the Page Source

```ruby
puts page.html
puts page.body
```

### Pause Execution

```ruby
# Use binding.pry or byebug — do NOT use sleep
binding.pry  # requires pry-byebug gem
```

Never use `sleep` to wait for elements. Capybara has a built-in retry mechanism with a configurable timeout. `sleep` creates brittle tests that fail on slow machines.

Configure the default wait time in test setup:

```ruby
# spec/support/capybara.rb
Capybara.default_max_wait_time = 5  # seconds; default is 2
```

If an element is not appearing, check that your app actually renders it — do not increase wait time as a fix.

---

## Running System Tests

```bash
# Minitest
bin/rails test:system
bin/rails test:system test/system/articles_test.rb

# Run with a visible browser for debugging
HEADLESS=false bin/rails test:system

# RSpec
bundle exec rspec spec/system/
bundle exec rspec spec/system/articles_spec.rb
```

Exclude system tests from the default test run (they are already excluded by `bin/rails test`):

```bash
# This does NOT run system tests
bin/rails test

# This runs everything
bin/rails test:all
```
