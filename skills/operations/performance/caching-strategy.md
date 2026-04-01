# Caching Strategy

Caching stores the result of expensive work and serves the stored result on repeat requests. Cache at the right layer, invalidate correctly, and you avoid most performance problems. Cache wrong and you serve stale data or waste memory.

## When to Cache

Cache when:
- The same data is fetched on many requests with the same result
- The data is expensive to compute (complex query, external API, heavy serialization)
- Stale-by-a-few-seconds is acceptable

Do not cache when:
- The data changes on every request (user-specific counters, real-time feeds)
- The data must always be current (payment status, inventory count)
- The computation is cheap and the cache adds more complexity than it saves
- You have not fixed N+1 queries yet — caching a bad query hides the problem

## Cache Layers (outermost to innermost)

### 1. HTTP Caching (free, no infrastructure required)

Leverage browser and CDN caches with response headers.

```ruby
# Public cache — CDN and browser can cache
def show
  @article = Article.find(params[:id])
  expires_in 10.minutes, public: true
end

# Conditional GET — returns 304 if content unchanged
def show
  @article = Article.find(params[:id])
  fresh_when(last_modified: @article.updated_at, etag: @article)
end

# Stale check with explicit freshness
def show
  @article = Article.find(params[:id])
  if stale?(@article)
    render json: @article
  end
  # if not stale, Rails auto-renders 304
end
```

### 2. Fragment Caching (view-level)

Cache portions of rendered HTML.

```erb
<%# Cache the entire list, busted when any post changes %>
<% cache ["posts-index", Post.maximum(:updated_at)] do %>
  <%= render @posts %>
<% end %>

<%# Cache individual items — Russian doll caching %>
<% @posts.each do |post| %>
  <% cache post do %>
    <%= render post %>
  <% end %>
<% end %>
```

Enable fragment caching in development to test:

```ruby
# config/environments/development.rb
config.action_controller.perform_caching = true
config.cache_store = :memory_store
```

### 3. Low-Level Caching (Rails.cache)

Cache arbitrary Ruby objects.

```ruby
# Basic read/write
Rails.cache.write("user:#{user.id}:score", 42, expires_in: 1.hour)
score = Rails.cache.read("user:#{user.id}:score")

# fetch — read or compute and store
trending = Rails.cache.fetch("posts:trending", expires_in: 15.minutes) do
  Post.trending.includes(:author).limit(20).to_a
end

# fetch_multi — batch read with single round-trip
post_ids = [1, 2, 3, 4, 5]
posts = Rails.cache.fetch_multi(*post_ids.map { |id| "post:#{id}" }, expires_in: 1.hour) do |key|
  id = key.split(":").last.to_i
  Post.find(id)
end
```

### 4. Query Caching (automatic, per request)

Rails caches identical SQL queries within a single request automatically. No configuration needed. The cache is cleared at the end of the request.

```ruby
# These fire only 1 SQL query total:
user = User.find(1)
user = User.find(1)  # served from request cache
```

## Cache Store Configuration

```ruby
# config/environments/production.rb

# Redis (recommended for multi-server deployments)
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  expires_in: 1.hour,
  error_handler: -> (method:, returning:, exception:) {
    Sentry.capture_exception(exception)
  }
}

# Memcache
config.cache_store = :mem_cache_store, "cache.example.com"

# Solid Cache (Rails 8 default — DB-backed, no Redis required)
config.cache_store = :solid_cache_store
```

## Invalidation Strategies

The hardest part of caching. Pick one strategy per cache entry and be consistent.

### Time-based (simplest)

```ruby
Rails.cache.fetch("analytics:dashboard", expires_in: 5.minutes) { compute_dashboard }
```

Acceptable when slightly stale data is fine and the data changes frequently anyway.

### Key-based (safest for correctness)

Embed a version into the cache key. When the data changes, the key changes, and the old entry is abandoned (expires naturally).

```ruby
# Model cache key includes updated_at
Rails.cache.fetch(["post", post.id, post.updated_at.to_i]) { expensive_serialization(post) }

# Rails builds this key for you — cache(post) uses post.cache_key_with_version
# post.cache_key_with_version => "posts/42-20240315123456789"
```

This is how `cache post` in ERB works. Add `touch: true` to associations so parent cache keys expire when children change:

```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true  # updates post.updated_at on comment change
end
```

### Explicit invalidation (most control, most risk)

Manually delete cache entries when data changes.

```ruby
class Post < ApplicationRecord
  after_commit :invalidate_cache

  private

  def invalidate_cache
    Rails.cache.delete("posts:featured")
    Rails.cache.delete_matched("posts:user:#{user_id}:*")
  end
end
```

Risk: forgetting to invalidate in all code paths. Prefer key-based invalidation where possible.

### Touch-based invalidation

Use `touch` to propagate updated_at up the association chain, which busts key-based caches automatically.

```ruby
class Tag < ApplicationRecord
  has_and_belongs_to_many :posts, after_add: :touch_post, after_remove: :touch_post

  private
  def touch_post(post)
    post.touch
  end
end
```

## Cache Key Design

Good cache keys are:
- **Unique** — different data → different key
- **Deterministic** — same data → same key on every server
- **Short** — Redis/Memcache have key length limits (~250 chars)

```ruby
# Include all dimensions that affect the output
cache_key = [
  "posts:index",
  current_user.role,           # different roles see different data
  params[:page],
  Post.maximum(:updated_at)    # bust when any post changes
].join(":")

Rails.cache.fetch(cache_key, expires_in: 5.minutes) { render_post_list }
```

Avoid interpolating user input directly into cache keys without sanitizing — it can create cache pollution.

## Monitoring Cache Effectiveness

```ruby
# Log cache hit/miss ratio
# In a Rails initializer:
ActiveSupport::Notifications.subscribe(/cache_.*\.active_support/) do |name, start, finish, id, payload|
  Rails.logger.debug "[CACHE] #{name} key=#{payload[:key]} #{(finish - start) * 1000}ms"
end
```

Check hit rate in production with Redis:

```bash
redis-cli INFO stats | grep keyspace
# keyspace_hits and keyspace_misses

# Hit rate = hits / (hits + misses) — aim for > 80%
```

Key metrics to track:

| Metric | Target | Action if off |
|---|---|---|
| Hit rate | > 80% | Increase TTL or improve key design |
| Eviction rate | Near 0% | Increase Redis memory |
| Cache size growth | Stable | Check for key proliferation |
| Avg fetch time | < 5ms | Check Redis network latency |
