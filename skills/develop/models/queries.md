# Queries

Active Record returns `ActiveRecord::Relation` objects from most query methods. Relations are lazy — they do not hit the database until you iterate, call `.to_a`, or use a terminal method like `.first`, `.count`, `.each`.

## Finders

### find

Retrieves by primary key. Raises `ActiveRecord::RecordNotFound` if not found.

```ruby
article = Article.find(1)
articles = Article.find(1, 2, 3)     # => [Article, Article, Article]
articles = Article.find([1, 2, 3])   # equivalent
```

### find_by

Returns the first match or `nil`. Raises with bang variant.

```ruby
article = Article.find_by(slug: "hello-world")
article = Article.find_by(title: "Hello", published: true)

article = Article.find_by!(slug: "hello-world")  # raises RecordNotFound if nil
```

### first / last / take

```ruby
Article.first         # ORDER BY id ASC LIMIT 1
Article.last          # ORDER BY id DESC LIMIT 1
Article.first(3)      # ORDER BY id ASC LIMIT 3  — returns array
Article.last(5)       # ORDER BY id DESC LIMIT 5 — returns array
Article.take          # no ORDER BY LIMIT 1 — fastest, arbitrary record
Article.take(3)       # no ORDER BY LIMIT 3
Article.first!        # raises ActiveRecord::RecordNotFound if no records
```

### all

Returns a relation representing all records. Lazy — no SQL until iterated.

```ruby
Article.all                    # SELECT * FROM articles
Article.all.to_a               # executes query, returns array
Article.all.count              # SELECT COUNT(*) FROM articles
```

## where Conditions

### Hash conditions

```ruby
Article.where(published: true)
Article.where(status: "active", published: true)

# IN clause from array
Article.where(status: ["draft", "review"])

# Range — translated to BETWEEN
Article.where(views_count: 100..500)
Article.where(created_at: 1.week.ago..)   # open-ended range (Ruby 2.6+)
```

### Array conditions (parameterized, safe from SQL injection)

```ruby
Article.where("title = ?", params[:title])
Article.where("published_at > ? AND views_count > ?", 1.week.ago, 100)
Article.where("title LIKE ?", "%#{query}%")

# Named placeholders
Article.where("status = :status AND author_id = :id", status: "active", id: author_id)
```

### where.not

```ruby
Article.where.not(published: false)
Article.where.not(status: nil)
Article.where.not(status: %w[draft archived])
```

### or

```ruby
Article.where(published: true).or(Article.where(featured: true))
```

### and (Rails 7+)

```ruby
published = Article.where(published: true)
featured  = Article.where(featured: true)
published.and(featured)   # WHERE published = TRUE AND featured = TRUE
```

## Ordering, Limiting, Offsetting

```ruby
Article.order(:created_at)                    # ASC by default
Article.order(created_at: :desc)
Article.order("created_at DESC, title ASC")

Article.limit(10)
Article.offset(20)
Article.limit(10).offset(20)                  # page 3 of 10

# Reorder (replace existing order)
Article.order(:title).reorder(:created_at)

# Remove order
Article.order(:title).unscoped.order(:created_at)
```

## Selecting Specific Columns

```ruby
Article.select(:id, :title, :published_at)
Article.select("title, LENGTH(title) AS title_length")

# pluck — returns plain array, no model instantiation (faster)
Article.pluck(:title)                  # => ["Hello", "World", ...]
Article.pluck(:id, :title)             # => [[1, "Hello"], [2, "World"], ...]
Article.where(published: true).pluck(:id)

# pick — pluck first result only (Rails 6+)
Article.where(slug: "hello").pick(:id)  # => 1

# ids — shorthand for pluck(:id)
Article.where(published: true).ids     # => [1, 2, 3]
```

## Scopes

Scopes are named, reusable query fragments defined on the model. They return a relation and chain with other query methods.

```ruby
class Article < ApplicationRecord
  scope :published,   -> { where(published: true) }
  scope :draft,       -> { where(published: false) }
  scope :recent,      -> { order(created_at: :desc) }
  scope :popular,     -> { where("views_count > ?", 1000) }
  scope :by_author,   ->(author) { where(author: author) }

  # default_scope applies to all queries — use sparingly
  default_scope -> { where(deleted_at: nil) }
end
```

```ruby
Article.published                          # WHERE published = TRUE
Article.published.recent                   # chained
Article.published.recent.limit(5)
Article.by_author(current_user).published

# Remove default_scope for a query
Article.unscoped.where(id: archived_ids)
```

Class methods that return a relation work identically to scopes and are preferred for complex logic:

```ruby
class Article < ApplicationRecord
  def self.search(query)
    return all if query.blank?
    where("title ILIKE :q OR body ILIKE :q", q: "%#{query}%")
  end
end

Article.published.search("rails")
```

## Joins

`joins` uses INNER JOIN — records without matching associations are excluded.

```ruby
# INNER JOIN using association name
Article.joins(:author)
Article.joins(:author, :comments)

# Nested joins
Article.joins(comments: :author)

# Custom SQL join
Article.joins("INNER JOIN authors ON authors.id = articles.author_id")
  .where(authors: { verified: true })

# Filter on joined table
Article.joins(:comments).where(comments: { spam: false }).distinct
```

`left_outer_joins` — include records even when there are no associated rows:

```ruby
# Authors with or without articles
Author.left_outer_joins(:articles)
      .select("authors.*, COUNT(articles.id) AS articles_count")
      .group("authors.id")
```

## Eager Loading (N+1 Prevention)

Loading associated records lazily causes an N+1 query problem:

```ruby
# BAD: 1 query for articles + 1 query per article for author = N+1
articles = Article.all
articles.each { |a| puts a.author.name }
```

Fix with eager loading:

### includes (recommended default)

Rails chooses the loading strategy automatically (usually two queries).

```ruby
# Two queries: one for articles, one for authors WHERE id IN (...)
articles = Article.includes(:author)
articles.each { |a| puts a.author.name }  # no extra queries

# Multiple associations
Article.includes(:author, :tags, :comments)

# Nested
Article.includes(comments: :author)

# With conditions on included association (forces LEFT OUTER JOIN)
Article.includes(:comments).where(comments: { approved: true }).references(:comments)
```

### preload

Always uses two separate queries. Cannot filter on the association.

```ruby
Article.preload(:author, :tags)
```

### eager_load

Always uses a single LEFT OUTER JOIN. Use when filtering on associated columns.

```ruby
Article.eager_load(:author).where(authors: { verified: true })
```

### Comparison table

| Method | SQL strategy | Filter on association? | When to use |
|---|---|---|---|
| `includes` | Rails decides (usually 2 queries) | Yes (with `references`) | Default — let Rails choose |
| `preload` | 2 queries | No | When you know you don't need to filter |
| `eager_load` | 1 LEFT OUTER JOIN | Yes | When filtering on associated table |
| `joins` | INNER JOIN | Yes | When filtering only, no loading |

### strict_loading

Raise an error if any association is lazily loaded (catches N+1 in development):

```ruby
# Per query
articles = Article.strict_loading.limit(10)

# Per record
article = Article.find(1)
article.strict_loading!
article.author   # raises ActiveRecord::StrictLoadingViolationError

# App-wide (config/environments/development.rb)
config.active_record.strict_loading_by_default = true
```

## Aggregations

```ruby
Article.count                            # SELECT COUNT(*) ...
Article.where(published: true).count
Article.count(:author_id)               # COUNT(author_id) — ignores NULLs

Article.sum(:views_count)
Article.average(:views_count)
Article.minimum(:views_count)
Article.maximum(:views_count)

# Group aggregations
Article.group(:status).count
# => { "draft" => 12, "published" => 45, "archived" => 7 }

Article.group(:author_id).sum(:views_count)
Article.group("DATE(created_at)").count

# Having (filter on aggregate)
Article.group(:author_id)
       .having("COUNT(*) > ?", 5)
       .count
```

## Batching Large Datasets

Never load millions of records into memory at once. Use batching:

### find_each

Yields one record at a time. Default batch size: 1000.

```ruby
Article.find_each do |article|
  ArticleIndexer.index(article)
end

# Options
Article.find_each(batch_size: 500, start: 1000, finish: 9999) do |article|
  # ...
end

# On a relation
Article.where(published: true).find_each(batch_size: 200) do |article|
  article.reindex
end
```

### find_in_batches

Yields arrays of records. Useful when you want to process a batch as a whole.

```ruby
Article.find_in_batches(batch_size: 1000) do |articles|
  Elasticsearch::Bulk.index(articles)
end
```

### in_batches (Rails 5+)

Yields relation objects. Useful for `update_all` or `delete_all` on batches.

```ruby
Article.where(published: false).in_batches(of: 500) do |batch|
  batch.update_all(archived: true)
end
```

Note: `find_each`/`find_in_batches` require an order on the primary key and will override any custom order clause.

## Arel

Arel is the query-building AST underneath Active Record. Access it directly for complex conditions you can't express cleanly with `where` strings.

```ruby
articles = Article.arel_table

# Greater than / less than
Article.where(articles[:views_count].gt(1000))
Article.where(articles[:created_at].lt(1.week.ago))

# Not equal
Article.where(articles[:status].not_eq("archived"))

# IN / NOT IN
Article.where(articles[:status].in(%w[draft review]))
Article.where(articles[:status].not_in(%w[archived deleted]))

# LIKE
Article.where(articles[:title].matches("%rails%"))

# Combine with and/or
Article.where(
  articles[:published].eq(true).and(
    articles[:views_count].gt(100).or(articles[:featured].eq(true))
  )
)
```

## Raw SQL

Use `find_by_sql` when you need full control:

```ruby
articles = Article.find_by_sql(
  "SELECT articles.*, authors.name AS author_name
   FROM articles
   INNER JOIN authors ON authors.id = articles.author_id
   WHERE articles.published = TRUE
   ORDER BY articles.created_at DESC
   LIMIT 20"
)

# With parameters (always use parameterized queries)
Article.find_by_sql(
  ["SELECT * FROM articles WHERE created_at > ?", 1.week.ago]
)
```

Execute without returning model objects:

```ruby
# Returns an array of hashes
result = ActiveRecord::Base.connection.execute("SELECT COUNT(*) FROM articles")

# Returns a result object
result = ActiveRecord::Base.connection.select_all("SELECT id, title FROM articles")
result.columns  # => ["id", "title"]
result.rows     # => [[1, "Hello"], [2, "World"]]
```

Sanitize values if you must interpolate:

```ruby
safe_value = ActiveRecord::Base.sanitize_sql_like(params[:search])
Article.where("title LIKE ?", "%#{safe_value}%")
```

## Existence Checks

```ruby
Article.exists?(1)                      # by PK
Article.exists?(title: "Hello")         # by attribute
Article.where(published: true).exists?  # on relation

# any? / none? / one? / many? — efficient (no full record load)
Article.where(published: true).any?     # SELECT 1 ... LIMIT 1
Article.where(published: true).none?
Article.where(author_id: 5).one?
Article.where(author_id: 5).many?
```

## Miscellaneous

```ruby
# Reload a record from the database
article.reload

# distinct / uniq
Article.joins(:tags).distinct

# Explain a query (outputs SQL EXPLAIN)
Article.where(published: true).explain

# Locking — pessimistic
Article.transaction do
  article = Article.lock.find(1)   # SELECT ... FOR UPDATE
  article.update!(views_count: article.views_count + 1)
end

# Locking — optimistic (requires lock_version column)
article = Article.find(1)
article.update!(title: "New Title")  # raises StaleObjectError if lock_version changed

# none — returns empty relation (useful for conditional query building)
scope :by_status, ->(s) { s.present? ? where(status: s) : none }
```
