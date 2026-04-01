# Multi-Tenancy

Multi-tenancy means one application instance serves multiple isolated customers (tenants). Each tenant's data must be invisible to other tenants. There are three isolation strategies: row-based, schema-based, and database-based.

## Choose a Strategy

```
How many tenants?
├── < 1,000 and/or each tenant needs custom schema → Schema-based (apartment)
└── Any other case → Row-based (acts_as_tenant) ← recommended default

Do tenants need absolute database-level isolation (regulated data, different DB servers)?
└── Yes → Database-based (complex — rarely needed)
```

## Row-Based Multi-Tenancy (Recommended)

Every tenant-scoped table has a `tenant_id` column. All queries include a `WHERE tenant_id = ?` condition. A library like `acts_as_tenant` automates scope enforcement.

### Install acts_as_tenant

```ruby
# Gemfile
gem "acts_as_tenant"
```

### Add tenant_id to every tenant-scoped table

```bash
bin/rails generate migration AddTenantIdToArticles tenant_id:integer:index
bin/rails db:migrate
```

### Create the Tenant model

```ruby
# app/models/organization.rb
class Organization < ApplicationRecord
  has_many :articles
  has_many :users
end
```

### Scope models to the tenant

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  acts_as_tenant :organization

  belongs_to :organization
  validates :title, presence: true
end
```

`acts_as_tenant` adds a default scope. `Article.all` becomes `Article.where(organization_id: current_tenant.id)` automatically. Creating a new record sets `organization_id` automatically.

### Set the current tenant per request

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  set_current_tenant_through_filter
  before_action :set_tenant

  private

  def set_tenant
    current_organization = Organization.find_by!(subdomain: request.subdomain)
    set_current_tenant(current_organization)
  end
end
```

### Pros and cons

| Pros | Cons |
|---|---|
| Simple to implement | Requires `tenant_id` on every table |
| Works with any database | One misbehaving query can leak data |
| Easy to query across tenants for admin/reporting | Default scopes can be surprising |
| Scales to thousands of tenants | |
| Standard Rails tooling — migrations, associations | |

## Schema-Based Multi-Tenancy

Each tenant gets a separate PostgreSQL schema within the same database. The `apartment` gem (or `apartment-activejob` for background jobs) switches the search path per request.

### Install

```ruby
# Gemfile
gem "apartment"
```

```bash
bundle install
bin/rails generate apartment:install
```

### Configure

```ruby
# config/initializers/apartment.rb
Apartment.configure do |config|
  # Models that live in the public schema (shared across tenants)
  config.excluded_models = %w[User Organization]

  # Use subdomain to identify tenants
  config.tenant_names = -> { Organization.pluck(:subdomain) }
end
```

### Create and switch tenants

```ruby
# Create a new schema for a new tenant
Apartment::Tenant.create("acme")

# Switch to a tenant schema
Apartment::Tenant.switch!("acme")
Article.all  # queries acme schema

# Switch back to public schema
Apartment::Tenant.reset
```

### Switch per request via middleware

```ruby
# config/application.rb — switch by subdomain automatically
config.middleware.use Apartment::Elevators::Subdomain
```

`apartment` hooks into `bin/rails db:migrate` and runs migrations in every tenant schema automatically.

### Pros and cons

| Pros | Cons |
|---|---|
| Strong schema-level isolation | PostgreSQL only |
| Each tenant can have custom schema changes | Managing hundreds of schemas is operationally complex |
| Easier to dump/restore a single tenant | Migrations must run N times (once per tenant) |
| No accidental data leaks from missing scope | Background jobs need tenant context threaded through |

## Database-Based Multi-Tenancy

Each tenant has a completely separate database. Switch `ActiveRecord::Base.establish_connection` per request. No gem abstracts this cleanly — implement it manually and accept the operational overhead.

Use only when tenants contractually require database-level isolation, run on different servers, or the tenant count is very small (tens, not thousands).

## Identify the Current Tenant

Three common strategies:

### Subdomain

```
acme.yourapp.com    → Organization.find_by(subdomain: "acme")
beta.yourapp.com    → Organization.find_by(subdomain: "beta")
```

```ruby
request.subdomain  # => "acme"
```

### URL path prefix

```
yourapp.com/acme/dashboard
yourapp.com/beta/dashboard
```

```ruby
# config/routes.rb
scope "/:organization_slug" do
  resources :articles
end

# In controller
Organization.find_by!(slug: params[:organization_slug])
```

### Custom domain

```
acme.com     → maps to acme tenant
beta-corp.io → maps to beta tenant
```

```ruby
organization = Organization.find_by!(custom_domain: request.host)
```

Requires DNS verification when tenants register custom domains.

## Test Multi-Tenancy Thoroughly

Always write tests that create two tenants and assert that one tenant cannot see the other's data. A missing scope is a silent data leak — it will not raise an error, it will just return wrong records.

```ruby
# test/models/article_test.rb
class ArticleTest < ActiveSupport::TestCase
  test "tenant cannot see another tenant's articles" do
    acme = organizations(:acme)
    beta = organizations(:beta)

    acme_article = Article.create!(title: "Acme Post", organization: acme)

    ActsAsTenant.with_tenant(beta) do
      assert_empty Article.all
      assert_raises(ActiveRecord::RecordNotFound) { Article.find(acme_article.id) }
    end
  end
end
```

Run these tests in CI. A data leak in a multi-tenant application is a critical security incident.

## Background Jobs

Background jobs run outside the request cycle — the tenant context is not set automatically. Pass the tenant ID as a job argument and restore it at the start of the job:

```ruby
# app/jobs/report_job.rb
class ReportJob < ApplicationJob
  def perform(organization_id)
    organization = Organization.find(organization_id)

    ActsAsTenant.with_tenant(organization) do
      # all queries inside here are scoped to organization
      Report.generate_for(organization)
    end
  end
end
```

Never store the tenant context in a thread-local variable outside of `with_tenant` — background job threads are reused across requests.
