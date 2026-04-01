# Database Tuning

Most Rails performance problems are database problems. Fix the database layer before adding caches or changing application code.

## Indexes

An index lets the database find rows without scanning every row in the table. Without an index on a `WHERE`, `ORDER BY`, or `JOIN` column, every query on that column is a full table scan.

### Add a single-column index

```ruby
# Migration
class AddIndexToUsersEmail < ActiveRecord::Migration[7.1]
  def change
    add_index :users, :email
  end
end
```

### Add a unique index

```ruby
add_index :users, :email, unique: true
add_index :posts, :slug, unique: true
```

Unique indexes enforce uniqueness at the database level, not just in Rails validations. Always add a unique index alongside `validates :field, uniqueness: true`.

### Add a composite index

Index multiple columns when queries filter on multiple columns together.

```ruby
# Query: WHERE user_id = ? AND published = true ORDER BY created_at DESC
add_index :posts, [:user_id, :published, :created_at]
```

Column order matters. The index is most useful when queries filter on a prefix of the indexed columns. Put the most selective column first.

### Add indexes in the generator

```bash
bin/rails generate migration AddStatusToOrders status:string:index
bin/rails generate model Post title:string user:references  # references adds index automatically
```

### Index foreign keys

Rails does not add indexes to foreign keys automatically (only `references` does). Check for missing FK indexes:

```sql
-- PostgreSQL: find foreign key columns without indexes
SELECT c.conrelid::regclass AS table,
       a.attname            AS column
FROM   pg_constraint c
JOIN   pg_attribute  a ON a.attrelid = c.conrelid
                      AND a.attnum = ANY(c.conkey)
WHERE  c.contype = 'f'
  AND  NOT EXISTS (
         SELECT 1 FROM pg_index i
         WHERE  i.indrelid = c.conrelid
           AND  a.attnum = ANY(i.indkey)
       );
```

## Finding Missing Indexes

### lol_dba gem

```ruby
group :development do
  gem "lol_dba"
end
```

```bash
bundle exec lol_dba db:find_indexes
```

Outputs a list of suggested `add_index` statements based on your schema.

### Check slow query log (PostgreSQL)

```sql
-- Enable slow query logging in postgresql.conf
log_min_duration_statement = 100  -- log queries over 100ms
```

Then grep your PostgreSQL logs for sequential scans.

## Reading EXPLAIN Output

Run `EXPLAIN ANALYZE` to see what PostgreSQL does for a query.

```ruby
# In Rails console
puts Post.where(user_id: 1).order(:created_at).explain
```

```sql
-- Run directly in psql
EXPLAIN ANALYZE SELECT * FROM posts WHERE user_id = 1 ORDER BY created_at DESC;
```

### What to look for

| Output term | Meaning | Action |
|---|---|---|
| `Seq Scan` | Full table scan | Add an index on the filter column |
| `Index Scan` | Using an index | Good |
| `Index Only Scan` | Using index, not reading table | Best — covering index |
| `Bitmap Heap Scan` | Index then batch row lookup | OK for larger result sets |
| `Hash Join` / `Merge Join` | Joining two tables | Usually fine, check row estimates |
| `rows=` estimate vs actual | Planner accuracy | Large gaps mean stale statistics — run ANALYZE |
| `cost=` | Planner's estimated cost | Higher = more expensive (arbitrary units) |

Run `ANALYZE` to refresh statistics:

```sql
ANALYZE posts;
ANALYZE;  -- analyze all tables
```

## Query Optimization

### Select only needed columns

```ruby
# BAD — loads all columns
User.all.map(&:email)

# GOOD — only the needed column
User.pluck(:email)           # returns Array of values
User.select(:id, :email)     # returns ActiveRecord::Relation with 2 columns
```

### Use pluck for scalar values

```ruby
# Returns array of emails, no model objects allocated
emails = User.where(active: true).pluck(:email)

# Multiple columns — returns array of arrays
pairs = User.pluck(:id, :email)  # [[1, "a@b.com"], [2, "c@d.com"]]
```

### Use pick for a single value

```ruby
# Rails 6+ — returns just the first value
email = User.where(id: 42).pick(:email)  # "user@example.com" or nil
```

### Use exists? instead of present?

```ruby
# BAD — loads an object to check existence
User.where(email: params[:email]).present?  # SELECT * LIMIT 1

# GOOD — SELECT 1 LIMIT 1
User.where(email: params[:email]).exists?
```

### Use find_each for batch processing

```ruby
# BAD — loads all records into memory
User.all.each { |user| user.send_newsletter }

# GOOD — loads in batches of 1000
User.find_each(batch_size: 1000) do |user|
  user.send_newsletter
end

# find_in_batches gives you the array for each batch
User.find_in_batches(batch_size: 500) do |batch|
  UserMailer.bulk(batch).deliver_later
end
```

### Avoid N+1 COUNT queries with counter caches

See `n-plus-one.md` for full counter cache setup.

```ruby
# BAD — COUNT query per post
posts.each { |p| p.comments.count }

# GOOD — reads cached column
posts.each { |p| p.comments.size }  # uses comments_count if counter cache enabled
```

### Use update_all and delete_all for bulk operations

```ruby
# BAD — loads records, fires N UPDATE queries
User.inactive.each { |u| u.update(deleted_at: Time.current) }

# GOOD — single SQL UPDATE
User.inactive.update_all(deleted_at: Time.current)

# GOOD — single SQL DELETE (bypasses callbacks)
User.inactive.where("last_seen_at < ?", 2.years.ago).delete_all
```

## Avoiding Full Table Scans

Full table scans occur when:
- There is no index on the filter column
- The query uses a function on an indexed column (defeats the index)
- The column has low selectivity (e.g., boolean) and no composite index

```ruby
# BAD — LOWER() defeats the index on email
User.where("LOWER(email) = ?", email.downcase)

# GOOD — store email downcased, or use a functional index
# Migration:
execute "CREATE INDEX index_users_on_lower_email ON users (LOWER(email));"
```

```ruby
# BAD — LIKE with leading wildcard cannot use a B-tree index
User.where("name LIKE ?", "%smith")

# For prefix search — B-tree works:
User.where("name LIKE ?", "smith%")

# For full-text search — use pg_search or tsvector
gem "pg_search"
```

## Connection Pooling

Rails uses a connection pool to share database connections across threads. Each web server process has its own pool.

### Configure pool size

```yaml
# config/database.yml
production:
  adapter: postgresql
  url: <%= ENV["DATABASE_URL"] %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000
  checkout_timeout: 5  # seconds to wait for a connection before raising
```

Match pool size to the number of threads your app server uses:

```
# Puma example: min_threads=5, max_threads=5 → pool: 5
# Puma example: max_threads=10 → pool: 10
```

Connections are a limited database resource. Do not set the pool too high — PostgreSQL has a `max_connections` limit (default 100).

### PgBouncer (production connection pooler)

For apps with many dynos/processes, use PgBouncer to multiplex thousands of app connections into fewer real database connections.

```
App server 1 (pool: 5) ──┐
App server 2 (pool: 5) ──┤──→ PgBouncer (20 real connections) ──→ PostgreSQL
App server 3 (pool: 5) ──┘
```

Set `pool` in database.yml to match PgBouncer's pool size per process, not total app threads.

## Common Antipatterns

| Antipattern | Fix |
|---|---|
| `User.all.count` | `User.count` (skips loading records) |
| `where(...).first` without `order` | Add explicit order or use `find_by` |
| `model.association.build` in a loop | Build outside the loop, use `insert_all` |
| Loading associations to check `.empty?` | Use `.exists?` or `.any?` with a subquery |
| `ORDER BY RANDOM()` | Use cursor-based pagination or app-level randomization |
| Joining tables without indexes on join columns | Add indexes on all FK and join columns |
| `select *` in subqueries | Select only the `id` column |
| Long transactions holding locks | Keep transactions tight; move non-DB work outside |

### insert_all and upsert_all (Rails 6+)

```ruby
# BAD — N individual INSERTs
records.each { |r| Model.create!(r) }

# GOOD — single bulk INSERT
Model.insert_all(records)                     # skip duplicates
Model.upsert_all(records, unique_by: :email)  # insert or update
```
