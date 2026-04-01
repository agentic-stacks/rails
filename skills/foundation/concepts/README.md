# Concepts: Rails Mental Model

Understand the core ideas that drive every Rails decision before writing a single line of code. These concepts recur in every other skill — return here when something feels arbitrary.

---

## Convention Over Configuration

Rails makes decisions for you. Follow naming conventions and you get behavior for free. Fight them and you fight the framework.

**The golden rule:** name things the Rails way and the wiring is automatic.

### Naming Convention Table

| What you name | Rails derives | Result |
|---|---|---|
| Model `User` | Table `users` | Active Record maps automatically |
| Model `BlogPost` | Table `blog_posts` | Underscored plural |
| `UsersController` | File `app/controllers/users_controller.rb` | Autoloaded by Zeitwerk |
| `Admin::UsersController` | File `app/controllers/admin/users_controller.rb` | Namespaced correctly |
| View for `UsersController#index` | `app/views/users/index.html.erb` | Rendered automatically |
| Helper for `UsersController` | `app/helpers/users_helper.rb` | Included automatically |
| Migration `CreateUsers` | File `db/migrate/TIMESTAMP_create_users.rb` | Run in timestamp order |

### When Conventions Break

```ruby
# Override table name when you must (legacy DB)
class User < ApplicationRecord
  self.table_name = "legacy_user_accounts"
end

# Override primary key
class User < ApplicationRecord
  self.primary_key = "user_uuid"
end
```

> WARNING: Overriding conventions is a signal that something external is forcing your hand (e.g., a legacy schema). Do it only when necessary — it adds cognitive overhead for every future developer.

---

## MVC Pattern

Rails organizes every application into three layers. Each layer has one job.

```
Browser
  │
  ▼
Router  (config/routes.rb)
  │  Maps URL + HTTP verb → controller action
  ▼
Controller  (app/controllers/)
  │  Authenticates, authorizes, calls models, assigns @variables
  ▼
Model  (app/models/)
  │  Validates data, runs business logic, queries database
  ▼
View  (app/views/)
  │  Renders HTML, JSON, or other formats using @variables
  ▼
Response → Browser
```

### Model

- Inherits from `ApplicationRecord` (which inherits from `ActiveRecord::Base`)
- Owns: validations, associations, scopes, business logic, callbacks
- Does NOT: receive HTTP parameters directly, render anything

```ruby
class Article < ApplicationRecord
  belongs_to :user
  validates :title, presence: true, length: { maximum: 255 }
  scope :published, -> { where(published: true) }

  def word_count
    body.split.length
  end
end
```

### View

- ERB templates in `app/views/<controller>/<action>.html.erb`
- Only accesses instance variables set by the controller (`@article`)
- Does NOT: query the database directly, contain business logic
- For JSON APIs: use `jbuilder`, `render json:`, or a serializer gem

```erb
<%# app/views/articles/show.html.erb %>
<h1><%= @article.title %></h1>
<p><%= @article.body %></p>
```

### Controller

- Inherits from `ApplicationController` (which inherits from `ActionController::Base`)
- Owns: reading params, calling models, setting `@variables`, choosing response format
- Does NOT: contain business logic, query the database directly

```ruby
class ArticlesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_article, only: [:show, :edit, :update, :destroy]

  def index
    @articles = Article.published.order(created_at: :desc)
  end

  def show; end  # @article set by before_action

  private

  def set_article
    @article = Article.find(params[:id])
  end
end
```

---

## Don't Repeat Yourself (DRY)

Every piece of knowledge has one authoritative location. Duplication creates drift — two copies diverge and introduce bugs.

### Where to Extract Shared Logic

| What is duplicated | Extract to |
|---|---|
| Business logic across models | `app/models/concerns/` |
| Controller behavior (auth, etc.) | `app/controllers/concerns/` |
| View fragments used in multiple places | Partials (`app/views/shared/_foo.html.erb`) |
| Helper methods used in multiple views | `app/helpers/application_helper.rb` |
| Complex query logic | Model scope or query object in `app/queries/` |
| Data transformation / calculations | Service object in `app/services/` |

### When to Not Extract

Duplication is sometimes correct. Extract only when:

1. The same logic appears in **3+ places** (rule of three), OR
2. The logic is **conceptually identical** (not just syntactically similar)

```
Is the code duplicated?
├── Yes, in 1-2 places → Wait. Duplication may be accidental similarity.
├── Yes, in 3+ places  → Extract to shared location.
└── No                 → Leave it.

Is this duplication conceptually the same thing?
├── Yes → Extract. One change should update all uses.
└── No  → Keep separate. Different concepts that look similar today
           may diverge tomorrow.
```

---

## Rails Philosophy

### Omakase

Rails is opinionated software. The name "Omakase" (Japanese: "I'll leave it to you") describes the philosophy: Rails selects the stack, you focus on your product. The choices are made by experienced practitioners. Learn why they exist before overriding them.

Defaults included in a Rails app:

- **Database:** SQLite (dev/test), PostgreSQL/MySQL in production
- **Frontend:** Hotwire (Turbo + Stimulus) over SPAs
- **Asset pipeline:** Propshaft + import maps (Rails 8), Sprockets (Rails 7 default)
- **Background jobs:** Solid Queue (Rails 8), adapt to Sidekiq/GoodJob for Rails 7
- **Caching:** Solid Cache (Rails 8), memory/Redis store for Rails 7
- **WebSockets:** Solid Cable (Rails 8), Action Cable with Redis adapter for Rails 7
- **Deployment:** Kamal (Rails 8), Capistrano or platform-managed for Rails 7

### Majestic Monolith

Rails advocates for a single, well-organized application over distributed microservices. Microservices add operational complexity (distributed tracing, service discovery, network latency, consistency) that is inappropriate for most applications. Start as a monolith. Extract services only when a specific bounded context has proven independent scaling requirements.

### Server-Rendered HTML (Hotwire)

Rails 7+ defaults to Hotwire rather than a JavaScript SPA framework. Turbo handles page navigation and partial updates without full-page reloads. Stimulus adds targeted JavaScript behavior. This keeps most logic server-side where Rails is strongest.

```
Choose Hotwire when:
  - You control the rendering stack
  - Forms, CRUD, dashboards, admin interfaces
  - You want Rails to own the business logic

Choose a SPA (React, Vue) when:
  - You need a rich offline-capable app
  - The client has complex local state
  - A separate mobile app already owns the API
```

---

## Rails 7 vs Rails 8: Philosophy Shifts

Rails 8 doubles down on simplicity by removing external dependencies that previously required Redis, Node, or additional infrastructure.

### Infrastructure Changes

| Concern | Rails 7 | Rails 8 |
|---|---|---|
| Background jobs | Sidekiq/GoodJob (Redis) | Solid Queue (SQLite/PG — no Redis) |
| Caching | Redis cache store | Solid Cache (DB-backed) |
| WebSocket adapter | Action Cable + Redis | Solid Cable (DB-backed) |
| Asset pipeline | Sprockets (default) | Propshaft (simpler, faster) |
| Deployment | Capistrano / platform PaaS | Kamal (Docker-based, built-in) |
| Authentication | Devise or roll-your-own | `rails generate authentication` (built-in generator) |

### What This Means in Practice

- A Rails 8 app can run in production on a single server with no Redis, no Node, and no external services beyond a database.
- Sprockets is still available in Rails 8 but not the default. New apps use Propshaft.
- Kamal ships with Rails 8 as the official deployment tool. It uses Docker and deploys to any VPS.
- Solid Queue stores jobs in the database. For high-throughput job queues (>1000 jobs/sec), Redis-backed alternatives remain better choices.

### Compatibility Decision Tree

```
Are you starting a new app?
├── Yes → Use Rails 8 defaults (Solid Queue/Cache/Cable, Propshaft, Kamal)
└── No  (existing Rails 7 app)
    ├── Running Redis already? → Keep existing adapters; upgrade when ready
    └── No Redis?
        └── Upgrading to Rails 8? → Migrate to Solid Queue/Cache/Cable
```

---

## Summary: The Three Questions

Before writing any Rails code, ask:

1. **What does the convention say?** — Check naming, directory placement, and inheritance chain first.
2. **Which layer owns this?** — Business logic in models, coordination in controllers, presentation in views.
3. **Is this already somewhere?** — Check concerns, helpers, and existing abstractions before writing new code.

## Next Steps

- Understand where files live: `skills/foundation/app-structure/`
- Configure environments and credentials: `skills/foundation/configuration/`
- Start building models: `skills/develop/models/`
