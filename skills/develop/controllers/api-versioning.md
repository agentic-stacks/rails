# API Versioning

Versioning lets you evolve an API without breaking existing clients. Choose a strategy before your API goes public — retrofitting versioning is expensive.

## Strategy Decision Tree

```
Do clients you control (mobile apps, SPAs) need to be updated when the API changes?
├── Yes (you can coordinate deploys and force upgrades)
│   └── URL versioning is simpler to implement and debug ──► see URL-Based Versioning
└── No (third-party clients, long support windows)
    └── Do clients need to request specific representations of the same resource?
        ├── Yes ──► Header versioning (Accept header) ──► see Header-Based Versioning
        └── No  ──► URL versioning is still fine for most cases
```

## URL-Based Versioning

The version is part of the URL path: `/api/v1/articles`. This is the most common approach — easy to understand, easy to test in a browser, trivially cache-able.

### Route setup

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :articles
      resources :users
    end

    namespace :v2 do
      resources :articles
    end
  end
end
```

Generated URLs:

```
GET /api/v1/articles      => Api::V1::ArticlesController#index
GET /api/v1/articles/:id  => Api::V1::ArticlesController#show
GET /api/v2/articles      => Api::V2::ArticlesController#index
```

### Directory structure

```
app/controllers/
  api/
    v1/
      articles_controller.rb
      users_controller.rb
      base_controller.rb
    v2/
      articles_controller.rb
      base_controller.rb
    base_controller.rb
```

### Base controllers

Create a base controller at each namespace level for shared behaviour:

```ruby
# app/controllers/api/base_controller.rb
module Api
  class BaseController < ActionController::API
    rescue_from ActiveRecord::RecordNotFound, with: :not_found

    private

    def not_found(e)
      render json: { error: e.message }, status: :not_found
    end
  end
end
```

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < Api::BaseController
      before_action :authenticate_request!

      private

      def authenticate_request!
        # v1 auth logic
      end
    end
  end
end
```

```ruby
# app/controllers/api/v1/articles_controller.rb
module Api
  module V1
    class ArticlesController < BaseController
      def index
        @articles = Article.order(created_at: :desc)
        render json: @articles
      end

      def show
        @article = Article.find(params[:id])
        render json: @article
      end
    end
  end
end
```

## Header-Based Versioning

The version is specified in the `Accept` header rather than the URL. Resources keep stable URLs while clients request specific representations:

```
Accept: application/vnd.myapp.v2+json
```

### Route constraint

Implement a constraint class that reads the `Accept` header:

```ruby
# app/constraints/api_version_constraint.rb
class ApiVersionConstraint
  def initialize(version:, default: false)
    @version = version
    @default = default
  end

  def matches?(request)
    return true if @default

    request.headers["Accept"].include?("application/vnd.myapp.v#{@version}+json")
  end
end
```

### Apply the constraint

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    scope module: :v2, constraints: ApiVersionConstraint.new(version: 2) do
      resources :articles
    end

    scope module: :v1, constraints: ApiVersionConstraint.new(version: 1, default: true) do
      resources :articles
    end
  end
end
```

Routes are matched top-to-bottom. v2 is checked first; v1 acts as the default fallback.

### Client usage

```bash
# Request v1 (default — no header required)
curl https://api.example.com/api/articles

# Request v2 explicitly
curl -H "Accept: application/vnd.myapp.v2+json" https://api.example.com/api/articles
```

### Caveats

- Harder to test in a browser or with basic `curl` without custom headers
- CDN and proxy caches may not vary correctly on `Accept` unless you set `Vary: Accept` in responses
- Middleware and logging must be configured to record the version

Add the `Vary` header in a base controller:

```ruby
class Api::BaseController < ActionController::API
  before_action { response.set_header("Vary", "Accept") }
end
```

## Sharing Code Between Versions

Avoid copy-pasting controller logic between versions. Keep shared query and serialisation logic in models, service objects, or concerns.

### Inherit from the previous version

When v2 only changes a few actions, inherit from v1 and override:

```ruby
# app/controllers/api/v2/articles_controller.rb
module Api
  module V2
    class ArticlesController < Api::V1::ArticlesController
      # Override only what changed in v2
      def index
        @articles = Article.published.order(published_at: :desc)
        render json: @articles
      end
    end
  end
end
```

### Service objects / query objects

Extract business logic into plain Ruby objects that any version can call:

```ruby
# app/services/articles/search.rb
module Articles
  class Search
    def initialize(params)
      @params = params
    end

    def call
      scope = Article.published
      scope = scope.where("title ILIKE ?", "%#{@params[:q]}%") if @params[:q]
      scope.order(published_at: :desc)
    end
  end
end

# Both v1 and v2 controllers use the same service
def index
  @articles = Articles::Search.new(params).call
  render json: @articles
end
```

### Version-specific serialisers

Use separate serialiser classes when the response shape differs:

```ruby
# v1: simple flat structure
class Api::V1::ArticleSerializer < ActiveModel::Serializer
  attributes :id, :title, :body
end

# v2: richer structure with relationships
class Api::V2::ArticleSerializer < ActiveModel::Serializer
  attributes :id, :title, :body, :slug, :published_at
  belongs_to :author
  has_many :tags
end
```

In the controller, specify the serialiser explicitly:

```ruby
render json: @article, serializer: Api::V2::ArticleSerializer
```

## Deprecating Old Versions

Signal to clients that a version is being retired before removing it.

### Add a deprecation header

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < Api::BaseController
      SUNSET_DATE = "2026-12-31"

      before_action :add_deprecation_headers

      private

      def add_deprecation_headers
        response.set_header("Sunset", SUNSET_DATE)
        response.set_header(
          "Deprecation",
          "The v1 API is deprecated. Migrate to v2 before #{SUNSET_DATE}."
        )
        response.set_header("Link", '<https://docs.example.com/api/v2>; rel="successor-version"')
      end
    end
  end
end
```

### Deprecation timeline

| Phase | Action |
|---|---|
| Announce | Publish sunset date in docs and via `Sunset` / `Deprecation` headers |
| Monitor | Track v1 traffic; contact high-volume clients |
| Soft sunset | Return 410 Gone for infrequently used endpoints |
| Hard sunset | Remove routes and controllers; update docs |

### Return 410 Gone on sunset date

```ruby
before_action :enforce_sunset

def enforce_sunset
  if Date.today >= Date.parse(SUNSET_DATE)
    render json: {
      error: "This API version has been discontinued. See https://docs.example.com/api/v2"
    }, status: :gone
  end
end
```

## URL vs Header Versioning: Comparison

| Dimension | URL (`/api/v1/`) | Header (`Accept: vnd...`) |
|---|---|---|
| Discoverability | Immediate — visible in URL | Requires reading docs |
| Browser testable | Yes | No (need curl or client) |
| Caching | Simple — URL uniquely identifies | Requires `Vary: Accept` |
| Client migration | Change base URL | Change header value |
| REST purity | Debated — resource URL changes | More RESTful by theory |
| Ecosystem support | Universal | Inconsistent |

For most teams, URL versioning is the pragmatic choice. Use header versioning only when strict URL stability is a hard requirement.
