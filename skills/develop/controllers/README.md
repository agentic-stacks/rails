# Controllers

Controllers are the C in MVC. They receive HTTP requests, orchestrate models, and hand data to views (or render JSON directly). Every public method in a controller that maps to a route is an **action**.

## Request Lifecycle

```
Browser
  └─► DNS resolution
        └─► Web server (Nginx / Puma)
              └─► Rack middleware stack
                    └─► Rails router (config/routes.rb)
                          └─► Controller#action
                                ├─► Model (ActiveRecord queries)
                                └─► View (ERB / Jbuilder / JSON)
                                      └─► Response (HTML / JSON / etc.)
```

Key stops:

| Stop | What happens |
|---|---|
| Web server | TLS termination, static file serving, request buffering |
| Rack middleware | Cookie parsing, session store, logging, CSRF protection |
| Router | Matches path + verb → dispatches to `Controller#action` |
| Controller | Authorises, fetches data, calls models, selects renderer |
| View / Renderer | Builds the response body |
| Response | Status code, headers, body sent back to the browser |

## Controller Naming

Controllers are Ruby classes that inherit from `ApplicationController`:

```ruby
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  # actions go here
end
```

Convention: plural resource name + `Controller`. File name matches the underscored class name.

| Class name | File path |
|---|---|
| `ArticlesController` | `app/controllers/articles_controller.rb` |
| `Admin::ArticlesController` | `app/controllers/admin/articles_controller.rb` |
| `Api::V1::UsersController` | `app/controllers/api/v1/users_controller.rb` |

## RESTful Actions

Rails maps seven standard actions to HTTP verbs and paths:

| Action | Verb | Path | Purpose |
|---|---|---|---|
| `index` | GET | `/articles` | List all records |
| `show` | GET | `/articles/:id` | Show one record |
| `new` | GET | `/articles/new` | Render creation form |
| `create` | POST | `/articles` | Persist new record |
| `edit` | GET | `/articles/:id/edit` | Render edit form |
| `update` | PATCH/PUT | `/articles/:id` | Persist changes |
| `destroy` | DELETE | `/articles/:id` | Delete record |

Not all controllers need all seven. Use `only:` or `except:` when declaring routes:

```ruby
resources :articles, only: [:index, :show]
```

## Generate a Controller

```bash
# Generate ArticlesController with index and show actions
bin/rails generate controller Articles index show

# What gets created:
# app/controllers/articles_controller.rb
# app/views/articles/index.html.erb
# app/views/articles/show.html.erb
# test/controllers/articles_controller_test.rb
# app/helpers/articles_helper.rb
```

Scaffold generates a full CRUD controller plus model and migration:

```bash
bin/rails generate scaffold Article title:string body:text published:boolean
bin/rails db:migrate
```

## ApplicationController

All controllers inherit from `ApplicationController`, which inherits from `ActionController::Base` (full stack) or `ActionController::API` (API-only apps).

Put shared behaviour — authentication, current user, error handling — in `ApplicationController`:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :authenticate_user!

  private

  def authenticate_user!
    redirect_to login_path unless current_user
  end

  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
end
```

## Files in This Skill

| File | What it covers |
|---|---|
| `routing.md` | `resources`, nested routes, namespaces, constraints, concerns |
| `actions.md` | Strong params, filters, rendering, redirects, flash, session |
| `api-mode.md` | API-only apps, serialisation, CORS, pagination, auth |
| `api-versioning.md` | URL-based and header-based versioning strategies |

## Next Steps

- Define routes: `routing.md`
- Write actions: `actions.md`
- Build an API: `api-mode.md`
- Version your API: `api-versioning.md`
