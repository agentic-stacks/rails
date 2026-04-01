# Migrations

Migrations are versioned, database-agnostic scripts that evolve your schema over time. Each migration is a Ruby class stored in `db/migrate/` with a timestamp prefix. Rails tracks which migrations have run in the `schema_migrations` table.

## Create a Migration

### Via model generator (preferred for new tables)

```bash
bin/rails generate model Article title:string body:text published:boolean
# Creates: db/migrate/20250101120000_create_articles.rb
```

### Via migration generator (for changes to existing tables)

Rails infers intent from the migration name:

```bash
# Add columns
bin/rails generate migration AddSlugToArticles slug:string:uniq
# => add_column :articles, :slug, :string + add_index

# Remove columns
bin/rails generate migration RemoveBodyFromArticles body:text
# => remove_column :articles, :body, :text

# Add a reference (foreign key + index)
bin/rails generate migration AddAuthorToArticles author:references
# => add_reference :articles, :author, null: false, foreign_key: true

# Join table for HABTM
bin/rails generate migration CreateJoinTableArticlesTags articles tags
# => create_join_table :articles, :tags
```

Name patterns Rails understands:

| Pattern | What Rails generates |
|---|---|
| `AddXxxToYyy` | `add_column` statements |
| `RemoveXxxFromYyy` | `remove_column` statements |
| `CreateYyy` | `create_table` block |
| `CreateJoinTableXxxYyy` | `create_join_table` block |
| `AddIndexToYyy` | `add_index` statement |

## Migration Anatomy

### The `change` method (preferred)

Rails can automatically reverse most operations:

```ruby
class CreateArticles < ActiveRecord::Migration[8.0]
  def change
    create_table :articles do |t|
      t.string  :title,     null: false
      t.text    :body
      t.boolean :published, null: false, default: false
      t.integer :views_count, null: false, default: 0
      t.references :author, null: false, foreign_key: true

      t.timestamps  # adds created_at and updated_at
    end

    add_index :articles, :title
  end
end
```

### The `up`/`down` methods (for irreversible operations)

```ruby
class ChangeArticlesStatusColumn < ActiveRecord::Migration[8.0]
  def up
    add_column :articles, :status, :string, default: "draft"
    Article.update_all(status: "published")
    remove_column :articles, :published
  end

  def down
    add_column :articles, :published, :boolean, default: false
    Article.where(status: "published").update_all(published: true)
    remove_column :articles, :status
  end
end
```

### Using `reversible` (hybrid)

```ruby
class AddDetailsToProducts < ActiveRecord::Migration[8.0]
  def change
    add_column :products, :price_currency, :string

    reversible do |direction|
      direction.up   { execute "UPDATE products SET price_currency = 'USD'" }
      direction.down { } # nothing needed
    end
  end
end
```

## Column Options Reference

```ruby
create_table :products do |t|
  t.string   :sku,        null: false, limit: 50
  t.decimal  :price,      precision: 10, scale: 2
  t.integer  :stock,      null: false, default: 0
  t.string   :status,     null: false, default: "active"
  t.text     :description
  t.datetime :archived_at                          # nullable datetime
  t.jsonb    :metadata,   default: {}              # PostgreSQL only
  t.timestamps                                      # created_at, updated_at
end
```

## Column Type Reference

| Ruby type | PostgreSQL | MySQL | SQLite |
|---|---|---|---|
| `:string` | VARCHAR(255) | VARCHAR(255) | TEXT |
| `:text` | TEXT | LONGTEXT | TEXT |
| `:integer` | INTEGER | INT | INTEGER |
| `:bigint` | BIGINT | BIGINT | INTEGER |
| `:float` | FLOAT | FLOAT | REAL |
| `:decimal` | DECIMAL | DECIMAL | NUMERIC |
| `:boolean` | BOOLEAN | TINYINT(1) | INTEGER |
| `:date` | DATE | DATE | TEXT |
| `:datetime` | TIMESTAMP | DATETIME | TEXT |
| `:json` | JSON | JSON | TEXT |
| `:jsonb` | JSONB | — | — |
| `:uuid` | UUID | CHAR(36) | TEXT |

## Reversible Operations

Operations Rails can auto-reverse in `change`:

- `create_table` / `drop_table`
- `add_column` / `remove_column` (requires type specified for remove)
- `add_index` / `remove_index`
- `add_reference` / `remove_reference`
- `add_foreign_key` / `remove_foreign_key`
- `rename_column`
- `rename_table`
- `change_column_null`
- `change_column_default`

Operations that require explicit `up`/`down`:

- `change_column` (cannot auto-reverse type changes)
- `execute` (raw SQL)
- Any data mutation

Mark a migration as explicitly irreversible:

```ruby
def change
  remove_column :articles, :legacy_field, :string
  # If down is called, raise instead of silently failing:
end

def down
  raise ActiveRecord::IrreversibleMigration
end
```

## Running Migrations

```bash
# Run all pending migrations
bin/rails db:migrate

# Check migration status
bin/rails db:migrate:status
# Output:
#  Status   Migration ID    Migration Name
# --------------------------------------------------
#    up     20250101000001  CreateArticles
#    up     20250115000001  AddSlugToArticles
#   down    20250201000001  AddCategoryToArticles

# Migrate to a specific version
bin/rails db:migrate VERSION=20250115000001

# Migrate only a specific database (multi-db)
bin/rails db:migrate:primary
bin/rails db:migrate:secondary
```

## Rolling Back Migrations

```bash
# Roll back the most recent migration
bin/rails db:rollback

# Roll back N migrations
bin/rails db:rollback STEP=3

# Roll back to a specific version
bin/rails db:migrate VERSION=20250101000001

# Redo (rollback then re-run) the last migration
bin/rails db:migrate:redo

# Redo last N migrations
bin/rails db:migrate:redo STEP=2
```

## Schema File

Rails maintains `db/schema.rb` as a snapshot of the current schema. It is regenerated every time you run `db:migrate`.

### schema.rb (default)

```ruby
# db/schema.rb — generated file, do not edit directly
ActiveRecord::Schema[8.0].define(version: 2025_02_01_000001) do
  create_table "articles", force: :cascade do |t|
    t.string   "title",       null: false
    t.text     "body"
    t.boolean  "published",   default: false, null: false
    t.datetime "created_at",  null: false
    t.datetime "updated_at",  null: false
    t.index ["title"], name: "index_articles_on_title"
  end
end
```

### structure.sql (for database-specific features)

Switch when you use triggers, stored procedures, views, custom functions, or PostgreSQL extensions:

```ruby
# config/application.rb
config.active_record.schema_format = :sql
```

```bash
bin/rails db:schema:dump   # regenerates db/structure.sql
bin/rails db:schema:load   # loads schema into database
```

Commit whichever file your app uses (`schema.rb` or `structure.sql`) — never both.

## Database Setup Commands

```bash
# Create the database (reads config/database.yml)
bin/rails db:create

# Drop the database
bin/rails db:drop

# Load schema into a freshly created database (skip migrations)
bin/rails db:schema:load     # from schema.rb
bin/rails db:structure:load  # from structure.sql

# Reset: drop + create + load schema + seed
bin/rails db:reset

# Prepare: idempotent setup (safe in CI)
# Creates DB if missing, runs pending migrations, does not seed
bin/rails db:prepare

# Seed with db/seeds.rb
bin/rails db:seed
```

## Indexes

```ruby
# Single column
add_index :articles, :slug, unique: true

# Composite index
add_index :articles, [:author_id, :published_at]

# Partial index (PostgreSQL)
add_index :articles, :published_at, where: "published = true"

# Concurrent index (PostgreSQL — avoids table lock)
add_index :articles, :title, algorithm: :concurrently

# Remove
remove_index :articles, :slug
remove_index :articles, column: [:author_id, :published_at]
```

## Foreign Keys

```ruby
# Add with referential integrity enforced at the DB level
add_foreign_key :articles, :users, column: :author_id

# With cascade delete
add_foreign_key :comments, :articles, on_delete: :cascade

# Remove
remove_foreign_key :articles, :users
remove_foreign_key :articles, column: :author_id
```

## Best Practices

**One concern per migration.** A migration that both adds a column and populates it is harder to roll back cleanly. Split into separate migrations or use `reversible`.

**Always write reversible migrations.** If `db:rollback` would fail, either implement `down` explicitly or raise `ActiveRecord::IrreversibleMigration`.

**Never modify a committed migration.** Once a migration has been merged and run by other developers or production, treat it as immutable. Create a new migration to fix the issue.

**Avoid loading models in schema migrations.** `Article.update_all(...)` in a migration will break if the `Article` class is later renamed or the column is removed. Use SQL directly:

```ruby
# Bad — model may not exist in future
Article.where(status: nil).update_all(status: "draft")

# Good — raw SQL is stable
execute "UPDATE articles SET status = 'draft' WHERE status IS NULL"
```

**Keep data migrations separate.** For bulk data transformations, use `maintenance_tasks` gem or a standalone rake task rather than bundling them into schema migrations. This keeps migrations fast, reversible, and focused on schema shape.

**Annotate large tables.** Adding a column to a table with millions of rows may lock it. Use `lock_timeout` or online schema change tools (pt-online-schema-change, gh-ost, strong_migrations gem).

```ruby
# strong_migrations gem adds safety checks
class AddStatusToOrders < ActiveRecord::Migration[8.0]
  def change
    add_column :orders, :status, :string
    # strong_migrations will warn if this operation is unsafe
  end
end
```
