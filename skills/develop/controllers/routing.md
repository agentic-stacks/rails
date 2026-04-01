# Routing

Routes live in `config/routes.rb`. The router matches an incoming HTTP verb + path and dispatches to a controller action. Rails uses the first matching route, so order matters.

## Inspect Routes

```bash
# List all routes
bin/rails routes

# Filter routes by controller, action, or path pattern
bin/rails routes -g article

# Show routes for a specific controller
bin/rails routes -c articles

# Open the route inspector in the browser (development only)
# Visit http://localhost:3000/rails/info/routes
```

## resources (Full)

Declare all seven RESTful routes in one line:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :articles
end
```

Generated routes:

| Helper | Verb | Path | Controller#Action |
|---|---|---|---|
| `articles_path` | GET | `/articles` | `articles#index` |
| `new_article_path` | GET | `/articles/new` | `articles#new` |
| `article_path(:id)` | GET | `/articles/:id` | `articles#show` |
| | POST | `/articles` | `articles#create` |
| `edit_article_path(:id)` | GET | `/articles/:id/edit` | `articles#edit` |
| | PATCH/PUT | `/articles/:id` | `articles#update` |
| | DELETE | `/articles/:id` | `articles#destroy` |

## resources (Partial)

Restrict which actions are generated:

```ruby
# Only the listed actions
resources :articles, only: [:index, :show]

# All actions except the listed ones
resources :articles, except: [:destroy]

# Single non-RESTful resource (no :id in path)
resource :profile   # GET /profile, PATCH /profile, etc.
```

## Nested Resources

Use nesting when one resource is owned by another:

```ruby
resources :articles do
  resources :comments
end
# => GET /articles/:article_id/comments
# => POST /articles/:article_id/comments
# => GET /articles/:article_id/comments/:id
```

Access the parent ID in the controller via `params[:article_id]`.

Limit nesting depth to one level. Deeper nesting produces unwieldy URLs and controller complexity.

## Shallow Nesting

Shallow nesting keeps collection routes nested but member routes at the top level:

```ruby
resources :articles do
  resources :comments, shallow: true
end
```

Generated paths:

| Action | Path |
|---|---|
| `comments#index` | `/articles/:article_id/comments` |
| `comments#new` | `/articles/:article_id/comments/new` |
| `comments#create` | `/articles/:article_id/comments` |
| `comments#show` | `/comments/:id` |
| `comments#edit` | `/comments/:id/edit` |
| `comments#update` | `/comments/:id` |
| `comments#destroy` | `/comments/:id` |

Alternatively, use the `shallow do` block:

```ruby
shallow do
  resources :articles do
    resources :comments
    resources :tags
  end
end
```

## Namespaces

Use `namespace` when controllers live in a module subdirectory and URLs share the prefix:

```ruby
namespace :admin do
  resources :articles   # => Admin::ArticlesController
  resources :users      # => Admin::UsersController
end
# URLs: /admin/articles, /admin/users
# Files: app/controllers/admin/articles_controller.rb
```

## Scopes

Use `scope` to add a URL prefix without changing the controller module:

```ruby
scope "/api" do
  resources :articles   # => ArticlesController (no module prefix)
end
# URL: /api/articles
# Controller: ArticlesController (not Api::ArticlesController)
```

Combine `scope` options:

```ruby
scope "/admin", module: "admin", as: "admin" do
  resources :articles   # => Admin::ArticlesController, admin_articles_path
end
```

### Namespace vs Scope Decision Tree

```
Do you want the URL prefix AND the module prefix AND the named helper prefix?
├── Yes ──► namespace :admin { ... }
└── No
    ├── URL prefix only (no module change) ──► scope "/prefix" { ... }
    ├── Module prefix only (no URL change) ──► scope module: "admin" { ... }
    └── Named helper prefix only          ──► scope as: "admin" { ... }
```

## Constraints

Restrict routes to specific parameter formats or request attributes.

### Parameter Constraints

```ruby
# :id must match the pattern
resources :articles, constraints: { id: /\d{5}/ }

# Inline for a single route
get "/articles/:id", to: "articles#show", constraints: { id: /[A-Z]+/ }
```

### Request Constraints (subdomain, format, etc.)

```ruby
# Only match requests to the api subdomain
namespace :api, constraints: { subdomain: "api" } do
  resources :articles
end

# Inline
get "/admin", to: "admin#index", constraints: { subdomain: "admin" }
```

### Lambda / Class Constraints

```ruby
# Lambda form
get "/premium", to: "premium#index", constraints: ->(req) { req.ssl? }

# Class form — implement matches?(request)
class MobileConstraint
  def matches?(request)
    request.user_agent.include?("Mobile")
  end
end

constraints MobileConstraint.new do
  get "/home", to: "mobile#index"
end
```

## Concerns

Extract shared route declarations to avoid duplication:

```ruby
concern :reviewable do
  resources :reviews
end

concern :taggable do
  resources :tags
end

resources :articles, concerns: [:reviewable, :taggable]
resources :products, concerns: :reviewable

# Equivalent to:
# resources :articles do
#   resources :reviews
#   resources :tags
# end
```

Concerns accept options passed at include time:

```ruby
concern :pageable do |options|
  get "page/:num", action: :index, as: "page", **options
end

resources :articles, concerns: :pageable
```

## Member and Collection Routes

Add custom routes to an existing resource.

### Member Routes (operate on one record — require `:id`)

```ruby
resources :articles do
  member do
    post "publish"
    get  "preview"
  end
end
# => POST /articles/:id/publish  => articles#publish
# => GET  /articles/:id/preview  => articles#preview
```

Single-line shorthand:

```ruby
resources :articles do
  post :publish, on: :member
end
```

### Collection Routes (operate on the set — no `:id`)

```ruby
resources :articles do
  collection do
    get "search"
    get "archived"
  end
end
# => GET /articles/search   => articles#search
# => GET /articles/archived => articles#archived
```

## Root Route

Maps `GET /` to a controller action:

```ruby
root "articles#index"

# Equivalent to:
root to: "articles#index"
```

## Named Routes and Path Helpers

Every route generates path and URL helpers:

```ruby
articles_path          # => "/articles"
article_path(@article) # => "/articles/42"
new_article_path       # => "/articles/new"
edit_article_path(@article) # => "/articles/42/edit"

# _url variants include protocol and host:
articles_url           # => "https://example.com/articles"
```

Override the helper name with `as:`:

```ruby
get "/exit", to: "sessions#destroy", as: :logout
# => logout_path, logout_url
```
