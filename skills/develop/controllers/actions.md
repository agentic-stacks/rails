# Controller Actions

An action is a public method on a controller class that Rails calls when a matching route is hit. Actions read params, call models, and render or redirect.

## Strong Parameters

Always whitelist params before passing them to a model. This prevents mass-assignment vulnerabilities.

```ruby
class ArticlesController < ApplicationController
  def create
    @article = Article.new(article_params)
    if @article.save
      redirect_to @article
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def article_params
    params.require(:article).permit(:title, :body, :published)
  end
end
```

### Permitting nested attributes

```ruby
# Scalar nested hash
params.require(:user).permit(:name, address: [:street, :city, :zip])

# Array of scalars
params.require(:article).permit(:title, tag_ids: [])

# Array of hashes (use permit on each element)
params.require(:article).permit(:title, comments_attributes: [:body, :author_id])
```

### Rails 8: `params.expect`

`params.expect` is a safer alternative introduced in Rails 8. It raises `ActionController::ParameterMissing` when the structure doesn't match and prevents exploits around arrays of hashes:

```ruby
# Equivalent to params.require(:article).permit(:title, :body)
article_params = params.expect(article: [:title, :body])

# Nested hash
user_params = params.expect(user: [:name, { address: [:street, :city] }])

# Array of hashes
params.expect(article: [:title, comments_attributes: [[*Comment::PERMITTED]]])
```

Use `expect` for new code targeting Rails 8+. Use `require` + `permit` when supporting Rails 7.

## Filters (Callbacks)

Filters run before, after, or around each action. Define them in the controller or in `ApplicationController` to apply globally.

### before_action

```ruby
class ArticlesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_article, only: [:show, :edit, :update, :destroy]

  def show
    # @article already set
  end

  private

  def set_article
    @article = Article.find(params[:id])
  end

  def authenticate_user!
    redirect_to login_path unless current_user
  end
end
```

A `before_action` that redirects or renders halts the chain — later filters and the action do not run.

### after_action

Runs after the action completes but before the response is sent:

```ruby
class ApplicationController < ActionController::Base
  after_action :log_request

  private

  def log_request
    Rails.logger.info "#{controller_name}##{action_name} responded #{response.status}"
  end
end
```

### around_action

Wraps the action — use `yield` to invoke the action:

```ruby
class ApplicationController < ActionController::Base
  around_action :measure_time

  private

  def measure_time
    started = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield
  ensure
    elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - started
    Rails.logger.info "#{action_name} took #{elapsed.round(3)}s"
  end
end
```

### skip_before_action

Override a globally-applied filter in a specific controller:

```ruby
class SessionsController < ApplicationController
  skip_before_action :authenticate_user!, only: [:new, :create]
end
```

### Filter Scope Options

| Option | Example | Effect |
|---|---|---|
| `only:` | `only: [:show, :index]` | Run only for listed actions |
| `except:` | `except: [:destroy]` | Run for all except listed |
| _(none)_ | | Run for all actions |

## Rendering

When an action neither redirects nor explicitly renders, Rails renders the matching view template automatically. Override this with explicit `render` calls.

### Render a template

```ruby
def new
  @article = Article.new
  render :new              # renders app/views/articles/new.html.erb
end

def create
  @article = Article.new(article_params)
  unless @article.save
    render :new, status: :unprocessable_entity   # form with errors
  end
end
```

### Render JSON

```ruby
def show
  @article = Article.find(params[:id])
  render json: @article
end

# With custom structure
def index
  @articles = Article.all
  render json: { articles: @articles, total: @articles.count }
end

# With status
render json: { error: "Not found" }, status: :not_found
```

### Common Status Symbols

| Symbol | HTTP Code | Typical use |
|---|---|---|
| `:ok` | 200 | Successful GET |
| `:created` | 201 | Successful POST that created a record |
| `:no_content` | 204 | Successful DELETE (no body) |
| `:unprocessable_entity` | 422 | Validation failed |
| `:not_found` | 404 | Record not found |
| `:unauthorized` | 401 | Not authenticated |
| `:forbidden` | 403 | Authenticated but not authorised |
| `:see_other` | 303 | Redirect after POST (form submission) |

### render vs redirect_to

```
Did the user submit a form that failed validation?
├── Yes ──► render :new, status: :unprocessable_entity
│           (re-renders form with @article errors in place)
└── No (action succeeded) ──► redirect_to @article
                               (issues HTTP 302/303, browser makes a new GET)
```

## Redirects

```ruby
redirect_to @article                          # /articles/:id
redirect_to articles_path                     # /articles
redirect_to root_path                         # /
redirect_to "https://example.com"             # external

# With HTTP status (use 303 after POST to prevent duplicate submissions)
redirect_to @article, status: :see_other

# With a flash message
redirect_to @article, notice: "Article created."
redirect_to @article, alert: "Something went wrong."
```

## Flash

Flash messages persist for exactly one redirect cycle.

```ruby
# Set and redirect
flash[:notice] = "Saved successfully."
redirect_to @article

# Shorthand
redirect_to @article, notice: "Saved successfully."

# Display in layout
# app/views/layouts/application.html.erb
<% flash.each do |type, message| %>
  <div class="flash flash--<%= type %>"><%= message %></div>
<% end %>
```

Use `flash.now` when rendering (not redirecting) — the message is available only for the current request:

```ruby
def create
  @article = Article.new(article_params)
  unless @article.save
    flash.now[:alert] = "Could not save article."
    render :new, status: :unprocessable_entity
  end
end
```

## Session

The session stores data server-side (default: signed cookie). It persists across requests until cleared.

```ruby
# Store
session[:user_id] = user.id

# Read
current_user = User.find_by(id: session[:user_id])

# Delete one key
session.delete(:user_id)

# Clear everything (after logout)
reset_session
```

## Cookies

```ruby
# Plain cookie (visible to browser JS)
cookies[:locale] = "en"

# With options
cookies[:token] = { value: "abc123", expires: 1.hour, httponly: true }

# Permanent (20 years)
cookies.permanent[:locale] = "fr"

# Signed (tamper-evident, not encrypted)
cookies.signed[:user_id] = current_user.id

# Encrypted (tamper-evident and unreadable)
cookies.encrypted[:cart] = cart.to_json

# Delete
cookies.delete(:locale)
```

## rescue_from

Handle exceptions at the controller level rather than scattering begin/rescue blocks:

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :record_not_found
  rescue_from ActiveRecord::RecordInvalid,  with: :record_invalid
  rescue_from ActionController::ParameterMissing, with: :bad_request

  private

  def record_not_found(exception)
    render json: { error: exception.message }, status: :not_found
  end

  def record_invalid(exception)
    render json: { errors: exception.record.errors.full_messages },
           status: :unprocessable_entity
  end

  def bad_request(exception)
    render json: { error: exception.message }, status: :bad_request
  end
end
```

Use `Article.find!` (raises) rather than `Article.find_by` (returns nil) to trigger `rescue_from` automatically.

## Streaming

Send large files or dynamically-generated content without buffering the entire body in memory.

### Send a file from disk

```ruby
def download
  send_file Rails.root.join("private", "reports", "#{params[:id]}.pdf"),
            type: "application/pdf",
            disposition: "attachment"
end
```

### Send data from memory

```ruby
def export_csv
  csv_data = generate_csv(Article.all)
  send_data csv_data,
            filename: "articles-#{Date.today}.csv",
            type: "text/csv",
            disposition: "attachment"
end
```

### Live streaming (ActionController::Live)

```ruby
class EventsController < ApplicationController
  include ActionController::Live

  def stream
    response.headers["Content-Type"] = "text/event-stream"
    response.headers["Last-Modified"] = Time.now.httpdate

    10.times do |i|
      response.stream.write "data: event #{i}\n\n"
      sleep 1
    end
  ensure
    response.stream.close
  end
end
```

Live streaming requires a multi-threaded server (Puma) and is not compatible with `before_action` callbacks that read the response body.
