# Multi-Database

Rails 6.0+ supports multiple databases natively. Common use cases: read replicas to scale reads, separate analytical databases, horizontal sharding for very large datasets, and hard data isolation between tenants.

## When to Use Multiple Databases

```
Do you need read scalability?
├── Yes — add a read replica with automatic role switching
│
Do you have multiple bounded domains that should never share tables?
├── Yes — use separate databases, one per domain
│
Do you have a single table exceeding ~100M rows with no viable partition strategy?
├── Yes — consider horizontal sharding
│
Is your Rails app below tens of millions of records?
└── No — stick with a single database
```

## Configuration: database.yml

### Read Replica Setup

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

production:
  primary:
    <<: *default
    database: my_app_production
    username: app_user
    password: <%= ENV["DATABASE_PASSWORD"] %>
    host: primary-db.example.com

  primary_replica:
    <<: *default
    database: my_app_production
    username: app_readonly
    password: <%= ENV["REPLICA_PASSWORD"] %>
    host: replica-db.example.com
    replica: true   # marks this as a replica — cannot run migrations
```

### Two Independent Databases

```yaml
production:
  primary:
    <<: *default
    database: app_primary
    host: primary-db.example.com

  animals:
    <<: *default
    database: app_animals
    host: animals-db.example.com
    migrations_paths: db/animals_migrate   # separate migration directory
```

## Connecting Models

### ApplicationRecord with replica

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class

  connects_to database: {
    writing: :primary,
    reading: :primary_replica
  }
end
```

All models that inherit from `ApplicationRecord` will write to `:primary` and read from `:primary_replica`.

### Per-model database connection

```ruby
# Models in a separate database
class AnimalsRecord < ApplicationRecord
  self.abstract_class = true

  connects_to database: {
    writing: :animals,
    reading: :animals
  }
end

class Dog < AnimalsRecord
end

class Cat < AnimalsRecord
end
```

```ruby
Dog.all       # queries the animals database
Article.all   # queries the primary database
```

## Automatic Role Switching (Read Replica)

Rails can route GET requests to the replica and writes/POST/PUT/PATCH/DELETE to the primary automatically.

```ruby
# config/application.rb
config.active_record.database_selector = { delay: 2.seconds }
config.active_record.database_resolver = ActiveRecord::Middleware::DatabaseSelector::Resolver
config.active_record.database_resolver_context =
  ActiveRecord::Middleware::DatabaseSelector::Resolver::Session
```

The `delay` option ensures reads hit the primary for 2 seconds after a write, preventing stale-read problems caused by replication lag.

### Manual Role Switching

```ruby
# Force a block to use the writing (primary) role
ActiveRecord::Base.connected_to(role: :writing) do
  Article.create!(title: "Breaking News")
end

# Force a block to use the reading (replica) role
ActiveRecord::Base.connected_to(role: :reading) do
  articles = Article.published.limit(20)
end
```

### Checking the Current Role

```ruby
ActiveRecord::Base.current_role   # => :writing or :reading
```

## Preventing Writes to Replicas

Mark a model as read-only so any attempt to write raises:

```ruby
class ReportingRecord < ApplicationRecord
  self.abstract_class = true
  connects_to database: { writing: :reporting, reading: :reporting }
end

class SalesReport < ReportingRecord
  # intentionally no writes — raise on any save attempt
end
```

Or prevent writes at the connection level:

```ruby
ActiveRecord::Base.connected_to(role: :reading, prevent_writes: true) do
  # Any INSERT/UPDATE/DELETE raises ActiveRecord::ReadOnlyError
  Article.where(published: false)
end
```

## Migrations for Multiple Databases

Each database has its own migrations directory and its own schema file.

### Directory structure

```
db/
  migrate/                     # primary database migrations
  animals_migrate/             # animals database migrations
  schema.rb                    # primary schema
  animals_schema.rb            # animals schema (generated automatically)
```

### database.yml migration path

```yaml
animals:
  <<: *default
  database: app_animals
  migrations_paths: db/animals_migrate
```

### Generating migrations

```bash
# Primary database (default)
bin/rails generate migration CreateArticles title:string

# Named database
bin/rails generate migration CreateDogs name:string --database=animals
```

### Running migrations per database

```bash
# All databases
bin/rails db:migrate

# Specific database
bin/rails db:migrate:primary
bin/rails db:migrate:animals

# Rollback specific database
bin/rails db:rollback:animals STEP=2

# Status per database
bin/rails db:migrate:status:primary
bin/rails db:migrate:status:animals
```

## Horizontal Sharding

Sharding splits one logical database across multiple physical database instances based on a key (e.g., tenant ID, user ID range).

### Configuration

```yaml
# config/database.yml
production:
  primary:
    <<: *default
    database: app_shard_default

  primary_shard_one:
    <<: *default
    database: app_shard_one

  primary_shard_two:
    <<: *default
    database: app_shard_two
```

### Model setup

```ruby
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class

  connects_to shards: {
    default:   { writing: :primary,          reading: :primary },
    shard_one: { writing: :primary_shard_one, reading: :primary_shard_one },
    shard_two: { writing: :primary_shard_two, reading: :primary_shard_two }
  }
end
```

### Switching shards

```ruby
# Explicitly select a shard for a block
ActiveRecord::Base.connected_to(shard: :shard_one) do
  Article.where(tenant_id: 1).all
end

# Combine shard + role
ActiveRecord::Base.connected_to(shard: :shard_two, role: :reading) do
  Report.expensive_query
end
```

### Shard selection middleware or service

In practice, shard selection is typically done in a middleware or a `before_action`:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  around_action :select_shard

  private

  def select_shard
    shard = ShardResolver.for_tenant(current_tenant)
    ActiveRecord::Base.connected_to(shard: shard) { yield }
  end
end
```

## Connection Pool Tuning

Each database connection uses a thread from the pool. Size it appropriately.

```yaml
production:
  primary:
    pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
    checkout_timeout: 5   # seconds before raising ConnectionTimeoutError
```

With Puma (default Rails server):

```ruby
# config/puma.rb
threads_count = ENV.fetch("RAILS_MAX_THREADS", 5).to_i
threads threads_count, threads_count

# Rule of thumb: pool size >= puma thread count
# If you have 2 databases: pool per DB >= puma threads
```

## Checking Connections

```ruby
# All connection pools
ActiveRecord::Base.connection_handler.connection_pool_list

# Which database a model connects to
Article.connection_db_config.database
Dog.connection_db_config.host
```

## Common Gotchas

**Foreign keys cannot span databases.** If `articles.author_id` references a user in a different database, the database cannot enforce that constraint. Enforce integrity in application code instead.

**Transactions do not span databases.** `ActiveRecord::Base.transaction` only covers one database connection. For cross-database consistency, use eventual consistency or distributed transaction patterns.

**Replication lag.** Replicas are typically milliseconds behind the primary. If a user writes and then immediately reads, they may see stale data. Use the `delay` option in database selector, or force a read from primary for critical reads after writes.

**schema.rb is per-database.** Each database in database.yml generates its own schema file. Ensure you commit all of them.

**Migrations must be scoped.** Running `db:migrate` without specifying a database runs all migrations across all configured databases. Ensure migration files land in the correct directory.
