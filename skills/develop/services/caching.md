# Caching

Rails provides fragment caching, low-level key-value caching, and HTTP conditional GET helpers. All share the `Rails.cache` interface so you can swap the storage backend without touching application code.

## Enable Caching in Development

```bash
bin/rails dev:cache    # toggles caching on/off; creates tmp/caching-dev.txt
```

## Fragment Caching

Wrap a view partial or block in `cache` to store the rendered HTML:

```erb
<%# app/views/articles/show.html.erb %>
<% cache @article do %>
  <article>
    <h1><%= @article.title %></h1>
    <%= @article.body %>
  </article>
<% end %>
```

The cache key is derived from the object: `articles/42-20240315120000000000` (model name, id, `updated_at`). When `updated_at` changes the old entry is abandoned and a new one is written.

Cache a collection efficiently with `cache_many` (Rails 7.1+):

```erb
<% cache_many @articles do |article| %>
  <%= render article %>
<% end %>
```

## Russian Doll Caching

Nest cache blocks so outer caches expire when inner caches change:

```erb
<%# Outer: expires when @project changes %>
<% cache @project do %>
  <h1><%= @project.name %></h1>

  <%# Inner: expires when each @task changes %>
  <% @project.tasks.each do |task| %>
    <% cache task do %>
      <%= render task %>
    <% end %>
  <% end %>
<% end %>
```

For the outer cache to expire when a task changes, touch the parent:

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  belongs_to :project, touch: true
end
```

## Cache Keys

Rails calls `cache_key_with_version` on ActiveRecord objects. You can customise cache keys by defining `cache_key` on a model or by passing an explicit key string:

```erb
<%# Explicit key — include locale so each language gets its own entry %>
<% cache ["en", @article] do %>
  ...
<% end %>
```

Helper for complex keys:

```ruby
cache_key = ["dashboard", current_user.id, current_user.updated_at.to_i]
Rails.cache.fetch(cache_key, expires_in: 30.minutes) { build_dashboard }
```

## Low-Level Caching

Use `Rails.cache.fetch` for anything that is not a view fragment: computed values, API responses, expensive queries.

```ruby
# Returns cached value or executes the block and stores the result
stats = Rails.cache.fetch("site/stats", expires_in: 1.hour) do
  { users: User.count, articles: Article.published.count }
end
```

Write and read explicitly:

```ruby
Rails.cache.write("user/#{user.id}/score", score, expires_in: 24.hours)
score = Rails.cache.read("user/#{user.id}/score")

# Delete a specific entry
Rails.cache.delete("user/#{user.id}/score")

# Delete all entries matching a pattern (not supported by all stores)
Rails.cache.delete_matched("user/*/score")

# Fetch multiple keys in one round-trip
values = Rails.cache.fetch_multi("a", "b", "c") { |key| compute(key) }
```

## Cache Stores

Configure in `config/environments/*.rb`:

```ruby
# In-memory (single process — good for development, not for multi-process prod)
config.cache_store = :memory_store, { size: 64.megabytes }

# Solid Cache — database-backed, Rails 8 default
config.cache_store = :solid_cache_store

# Redis
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  expires_in: 90.minutes,
  error_handler: ->(method:, returning:, exception:) {
    Rails.logger.error "Redis cache error: #{exception}"
  }
}

# Memcached
config.cache_store = :mem_cache_store, "cache-1.example.com", "cache-2.example.com"

# Null store — disables caching (useful in tests)
config.cache_store = :null_store
```

## Solid Cache (Rails 8)

Solid Cache stores cache entries in a dedicated database table. No Redis required.

**Install:**

```bash
bin/rails solid_cache:install
bin/rails db:migrate
```

This creates a `solid_cache_entries` table and sets `config.cache_store = :solid_cache_store` in `config/environments/production.rb`.

Configure in `config/cache.yml`:

```yaml
# config/cache.yml
default: &default
  store_options:
    # Maximum cache size in bytes (default: 256 MB)
    max_size: <%= 256.megabytes %>
    # Number of database shards (advanced)
    # shards: 1

development:
  <<: *default

production:
  <<: *default
  store_options:
    max_size: <%= 1.gigabyte %>
```

## Conditional GET (HTTP Caching)

Tell browsers and proxies when a response has not changed, avoiding a full render:

```ruby
# app/controllers/articles_controller.rb
def show
  @article = Article.find(params[:id])

  # Returns 304 Not Modified if the client's cached version is still fresh
  if stale?(@article)
    respond_to do |format|
      format.html
      format.json { render json: @article }
    end
  end
end
```

Set cache headers explicitly:

```ruby
def index
  @articles = Article.published.order(created_at: :desc)
  fresh_when(etag: @articles, last_modified: @articles.maximum(:updated_at))
end
```

`stale?` and `fresh_when` both accept:
- An ActiveRecord object or collection
- `etag:` — string or object used to compute an ETag
- `last_modified:` — Time
- `public: true` — allows shared (CDN) caching

## Cache Invalidation

### Touch associations

```ruby
# Expire a project's cache when a task is saved
class Task < ApplicationRecord
  belongs_to :project, touch: true
end
```

### Manual delete in a callback

```ruby
class Article < ApplicationRecord
  after_commit :expire_cached_views

  private

  def expire_cached_views
    Rails.cache.delete("views/articles/#{id}")
  end
end
```

### Versioned keys

Append a version to the cache key so old entries are abandoned rather than deleted:

```ruby
CACHE_VERSION = "v2"

Rails.cache.fetch("dashboard/#{CACHE_VERSION}/#{user.id}") { ... }
```

Bump `CACHE_VERSION` to bust all entries atomically on next deploy.
