# N+1 Queries

An N+1 query is when loading a collection runs 1 query to get the records, then fires N additional queries — one per record — to fetch an association. The result is dozens or hundreds of tiny queries instead of one or two efficient ones.

## The Classic N+1

```ruby
# Controller
def index
  @posts = Post.all   # Query 1: SELECT * FROM posts
end
```

```erb
<%# View — fires one query per post %>
<% @posts.each do |post| %>
  <%= post.author.name %>   <%# Query 2, 3, 4 ... N+1 %>
<% end %>
```

```sql
-- What you get in the log:
SELECT * FROM posts
SELECT * FROM users WHERE id = 1
SELECT * FROM users WHERE id = 2
SELECT * FROM users WHERE id = 3
-- ... one per post
```

Fix it with `includes`:

```ruby
@posts = Post.includes(:author)  # 2 queries total regardless of count
```

```sql
SELECT * FROM posts
SELECT * FROM users WHERE id IN (1, 2, 3, ...)
```

## Bullet Gem Setup

Bullet detects N+1 queries, unused eager loading, and missing counter caches in development.

```ruby
group :development do
  gem "bullet"
end
```

```ruby
# config/environments/development.rb
config.after_initialize do
  Bullet.enable        = true
  Bullet.alert         = true          # JS alert in browser
  Bullet.bullet_logger = true          # log/bullet.log
  Bullet.console       = true          # browser JS console
  Bullet.rails_logger  = true          # Rails.logger

  # Raise in test so CI catches regressions:
  Bullet.raise         = Rails.env.test?

  # Optionally add to Rack:
  config.middleware.use Bullet::Rack
end
```

```ruby
# For RSpec — add to spec/rails_helper.rb
if Bullet.enable?
  config.before(:each) { Bullet.start_request }
  config.after(:each) do
    Bullet.perform_out_of_channel_notifications if Bullet.notification?
    Bullet.end_request
  end
end
```

Bullet adds entries to `log/bullet.log`:

```
USE eager loading detected
  Post => [:author]
  Add to your finder: :includes => [:author]
```

## includes, preload, and eager_load

These all prevent N+1 but work differently. Choose based on whether you need to filter on the association.

### includes (auto-selects strategy)

```ruby
# Rails decides: uses preload unless you reference the association in WHERE/ORDER
Post.includes(:author)
Post.includes(:author, :tags, comments: :author)
```

When you add a `where` or `order` on the association, Rails switches to a JOIN automatically:

```ruby
Post.includes(:author).where(authors: { verified: true })
# => uses JOIN (eager_load strategy)
```

### preload (always 2+ separate queries)

```ruby
# Always issues separate queries — safe for large associations
Post.preload(:author)
Post.preload(:comments)
```

Use `preload` when:
- The association table is very large and a JOIN would be slow
- You want to avoid a massive Cartesian product from joining multiple has_many associations

### eager_load (always a LEFT OUTER JOIN)

```ruby
# Joins everything into one query — allows WHERE/ORDER on association columns
Post.eager_load(:author).where(authors: { verified: true }).order("authors.name")
```

Use `eager_load` when:
- You need to filter or sort by association attributes
- You want a single query and the JOIN won't explode row count

### Summary

| Method | Strategy | Use when |
|---|---|---|
| `includes` | auto | Default — let Rails choose |
| `preload` | separate queries | Multiple has_many, avoiding row explosion |
| `eager_load` | LEFT OUTER JOIN | Filtering/sorting on association columns |

## strict_loading

Rails 6.1+ can raise an error when an association is lazy-loaded (i.e., not eagerly loaded). Use it to catch N+1 in tests.

```ruby
# Per query — raises if any association is lazy-loaded on results
Post.strict_loading.all

# Per model — always strict
class Post < ApplicationRecord
  self.strict_loading_by_default = true
end

# Per association
class User < ApplicationRecord
  has_many :posts, strict_loading: true
end
```

```ruby
# In test environment
# config/environments/test.rb
config.active_record.strict_loading_by_default = true
```

When an association is accessed without eager loading, Rails raises `ActiveRecord::StrictLoadingViolationError` instead of silently running a query.

## Counter Caches

Avoid count queries on associations by caching the count on the parent record.

```ruby
# Without counter cache — fires COUNT query every time
post.comments.count   # SELECT COUNT(*) FROM comments WHERE post_id = ?
```

Add a migration:

```bash
bin/rails generate migration AddCommentsCountToPosts comments_count:integer
```

```ruby
# Migration
class AddCommentsCountToPosts < ActiveRecord::Migration[7.1]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false

    # Backfill existing data
    Post.find_each do |post|
      Post.reset_counters(post.id, :comments)
    end
  end
end
```

```ruby
# Model — declare counter_cache on the belongs_to side
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end
```

```ruby
# Now this reads from the column — no query
post.comments.size    # reads posts.comments_count
post.comments.count   # still runs COUNT — use .size instead
```

## Common N+1 Patterns in Views

### Partials rendering collections

```erb
<%# BAD — each partial lazy-loads author %>
<%= render @posts %>

<%# GOOD — preload before rendering %>
<%# Controller: @posts = Post.includes(:author).all %>
<%= render @posts %>
```

### Nested associations

```ruby
# Controller
@posts = Post.includes(comments: :author).limit(20)
```

```erb
<% @posts.each do |post| %>
  <% post.comments.each do |comment| %>
    <%= comment.author.name %>   <%# no query — already loaded %>
  <% end %>
<% end %>
```

### Serializers (ActiveModel Serializers / fast_jsonapi)

```ruby
# BAD — serializer calls post.author for every post
class PostSerializer < ActiveModel::Serializer
  attributes :title
  belongs_to :author
end

# In controller — must eager load for the serializer
render json: PostSerializer.new(Post.includes(:author).all)
```

### Checking existence

```ruby
# BAD — loads entire association to check existence
post.comments.any?     # if comments not loaded: SELECT COUNT(*) or SELECT 1

# GOOD — use the counter cache or a targeted exists? call
post.comments.exists?  # SELECT 1 FROM comments WHERE post_id = ? LIMIT 1
post.comments.loaded? ? post.comments.any? : post.comments.exists?
```

### Polymorphic associations

Polymorphic associations cannot be eager loaded across types in a single query. Handle with care:

```ruby
# This will still cause N+1 per type:
@activities = Activity.includes(:trackable)

# Work around by grouping:
Activity.all.group_by(&:trackable_type).each do |type, activities|
  klass = type.constantize
  ids = activities.map(&:trackable_id)
  records = klass.where(id: ids).index_by(&:id)
  activities.each { |a| a.trackable = records[a.trackable_id] }
end
```
