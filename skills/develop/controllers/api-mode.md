# API Mode

Rails ships with a dedicated API mode that strips browser-oriented middleware and defaults every controller to `ActionController::API`. Use it when building a JSON backend that is consumed by a separate frontend, mobile client, or another service.

## Create an API-Only App

```bash
rails new my_api --api
```

What `--api` changes compared to a standard app:

| Aspect | Standard | API mode |
|---|---|---|
| Controller base class | `ActionController::Base` | `ActionController::API` |
| Middleware | Full stack (cookies, sessions, flash, CSRF) | Trimmed (no browser-only middleware) |
| Generators | Create views, helpers, assets | Skip views, helpers, assets |
| `config/application.rb` | `config.api_only` absent | `config.api_only = true` |

## Convert an Existing App to API Mode

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    config.api_only = true
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
end
```

Remove or comment out middleware entries that are browser-specific (e.g., `ActionDispatch::Cookies`, `ActionDispatch::Session::*`).

## ActionController::API vs ActionController::Base

`ActionController::API` includes a focused subset of modules:

| Module | Purpose |
|---|---|
| `UrlFor` | Route helpers (`articles_path`, etc.) |
| `Redirecting` | `redirect_to` |
| `Renderers::All` | `render json:`, `render xml:`, etc. |
| `ConditionalGet` | `stale?`, ETags, Last-Modified |
| `StrongParameters` | `params.require`, `params.permit` |
| `DataStreaming` | `send_file`, `send_data` |
| `Caching` | Action and fragment caching |

Notable omissions: cookies, sessions, flash, CSRF protection, layouts, view rendering. Add individual modules back if needed:

```ruby
class ApplicationController < ActionController::API
  include ActionController::Cookies   # add cookies back
end
```

## JSON Serialisation

### render json: (built-in, no gem required)

```ruby
def show
  @article = Article.find(params[:id])
  render json: @article
end

def index
  @articles = Article.all
  render json: @articles
end
```

Control which attributes are exposed by overriding `as_json` on the model or passing options:

```ruby
render json: @article.as_json(only: [:id, :title, :published_at])
render json: @article.as_json(except: [:created_at, :updated_at])
render json: @article.as_json(include: { author: { only: [:id, :name] } })
```

### Jbuilder

Jbuilder (included in Rails by default for full-stack apps, add the gem for API-only) uses view templates for fine-grained JSON control:

```ruby
# Gemfile
gem "jbuilder"
```

```ruby
# app/views/articles/show.json.jbuilder
json.id      @article.id
json.title   @article.title
json.body    @article.body
json.author do
  json.id   @article.author.id
  json.name @article.author.name
end
json.tags @article.tags, :id, :name
```

Call from the controller without specifying a renderer — Rails picks the `.json.jbuilder` template based on the request format:

```ruby
def show
  @article = Article.find(params[:id])
  # renders app/views/articles/show.json.jbuilder automatically
end
```

### Active Model Serializers (AMS)

AMS defines serialisation logic in a dedicated class:

```ruby
# Gemfile
gem "active_model_serializers"
```

```ruby
# app/serializers/article_serializer.rb
class ArticleSerializer < ActiveModel::Serializer
  attributes :id, :title, :body, :published_at
  belongs_to :author
  has_many :tags
end
```

```ruby
# Controller — AMS picks up the serialiser automatically
def show
  @article = Article.find(params[:id])
  render json: @article
end
```

### Choosing a Serialisation Strategy

```
Do you need view-level logic (conditionals, loops, ad-hoc structures)?
├── Yes ──► Jbuilder (template files, familiar ERB-like DSL)
└── No
    ├── Is the API surface small / greenfield?
    │   └── Yes ──► render json: with as_json overrides (zero dependencies)
    └── Do you have many models with consistent serialisation rules?
        └── Yes ──► Active Model Serializers or blueprinter gem
```

## Pagination

Never return unbounded collections. Paginate at the database level.

### kaminari

```ruby
# Gemfile
gem "kaminari"
```

```ruby
def index
  @articles = Article.order(created_at: :desc).page(params[:page]).per(params[:per_page] || 20)
  render json: {
    articles: @articles,
    meta: {
      current_page: @articles.current_page,
      total_pages:  @articles.total_pages,
      total_count:  @articles.total_count
    }
  }
end
```

### pagy

pagy is faster and uses less memory than kaminari for large datasets:

```ruby
# Gemfile
gem "pagy"
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include Pagy::Backend
end
```

```ruby
def index
  @pagy, @articles = pagy(Article.order(created_at: :desc), items: 20)
  render json: {
    articles: @articles,
    meta: pagy_metadata(@pagy)
  }
end
```

## CORS

Browsers block cross-origin requests by default. Configure CORS with the `rack-cors` gem:

```ruby
# Gemfile
gem "rack-cors"
```

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "https://app.example.com"   # exact origin

    resource "*",
      headers: :any,
      methods: [:get, :post, :patch, :put, :delete, :options, :head],
      credentials: true
  end
end
```

Development with multiple origins:

```ruby
origins Rails.env.production? ? "https://app.example.com" : "*"
```

Expose custom response headers (e.g., pagination headers):

```ruby
resource "*",
  headers: :any,
  expose: ["X-Total-Count", "X-Page"],
  methods: [:get, :post, :patch, :put, :delete, :options, :head]
```

## Authentication Patterns

### Token authentication (simple)

```ruby
class ApplicationController < ActionController::API
  before_action :authenticate_request!

  private

  def authenticate_request!
    token = request.headers["Authorization"]&.split(" ")&.last
    @current_user = User.find_by(api_token: token)
    render json: { error: "Unauthorized" }, status: :unauthorized unless @current_user
  end
end
```

### JWT

```ruby
# Gemfile
gem "jwt"
```

```ruby
class JsonWebToken
  SECRET = Rails.application.credentials.jwt_secret!

  def self.encode(payload, exp = 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET)
  end

  def self.decode(token)
    decoded = JWT.decode(token, SECRET)[0]
    HashWithIndifferentAccess.new(decoded)
  rescue JWT::DecodeError => e
    raise ActiveRecord::RecordNotFound, e.message
  end
end

class ApplicationController < ActionController::API
  before_action :authenticate_request!

  private

  def authenticate_request!
    header = request.headers["Authorization"]
    token  = header.split(" ").last if header
    decoded = JsonWebToken.decode(token)
    @current_user = User.find(decoded[:user_id])
  rescue ActiveRecord::RecordNotFound
    render json: { error: "Unauthorized" }, status: :unauthorized
  end
end
```

## Consistent Error Responses

Define a standard error shape and use `rescue_from` to enforce it:

```ruby
class ApplicationController < ActionController::API
  rescue_from ActiveRecord::RecordNotFound,       with: :not_found
  rescue_from ActiveRecord::RecordInvalid,        with: :unprocessable_entity
  rescue_from ActionController::ParameterMissing, with: :bad_request

  private

  def not_found(e)
    render_error(:not_found, e.message)
  end

  def unprocessable_entity(e)
    render_error(:unprocessable_entity, e.record.errors.full_messages)
  end

  def bad_request(e)
    render_error(:bad_request, e.message)
  end

  def render_error(status, detail)
    render json: {
      error: {
        status: Rack::Utils.status_code(status),
        detail: detail
      }
    }, status: status
  end
end
```

Error response shape:

```json
{
  "error": {
    "status": 422,
    "detail": ["Title can't be blank", "Body is too short"]
  }
}
```

Keeping the shape consistent lets API clients handle errors with a single code path.
