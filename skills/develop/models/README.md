# Active Record: Models

Active Record is Rails' ORM (Object-Relational Mapping) layer. It maps Ruby classes to database tables, table rows to Ruby objects, and table columns to object attributes. You write Ruby; Active Record handles the SQL.

## Naming Conventions

Rails derives table names from model class names automatically. Follow these conventions — override only when working with a legacy schema.

| Model class | Table name |
|---|---|
| `Article` | `articles` |
| `User` | `users` |
| `ArticleComment` | `article_comments` |
| `Person` | `people` |
| `Ox` | `oxen` |

Rails uses `ActiveSupport::Inflector` for pluralization. Check an edge case:

```bash
bin/rails console
> "ArticleComment".tableize   # => "article_comments"
> "Person".tableize           # => "people"
```

Override the convention for a single model:

```ruby
class LegacyOrder < ApplicationRecord
  self.table_name = "legacy_tbl_orders"
  self.primary_key = "order_id"
end
```

## Schema Conventions

Rails expects (and auto-manages) certain column names:

| Column | Purpose |
|---|---|
| `id` | Primary key (integer by default, bigint in Rails 8) |
| `created_at` | Set on insert — never touch manually |
| `updated_at` | Updated on every save — never touch manually |
| `type` | STI discriminator column — see `associations.md` |
| `<model>_id` | Foreign key (e.g., `author_id` on `books`) |
| `<model>_type` | Polymorphic type column (e.g., `imageable_type`) |

## Generate a Model

```bash
bin/rails generate model Article title:string body:text published:boolean views_count:integer

# Generates:
#   app/models/article.rb
#   db/migrate/20250101000000_create_articles.rb
#   test/models/article_test.rb
#   test/fixtures/articles.yml
```

Column type shorthands:

| Shorthand | SQL type |
|---|---|
| `string` | VARCHAR(255) |
| `text` | TEXT |
| `integer` | INTEGER |
| `bigint` | BIGINT |
| `float` | FLOAT |
| `decimal{10,2}` | DECIMAL(10,2) |
| `boolean` | BOOLEAN |
| `date` | DATE |
| `datetime` | DATETIME / TIMESTAMP |
| `references` / `belongs_to` | Foreign key + index |
| `string:index` | Column with index |
| `string!` | NOT NULL constraint |

```bash
# With a foreign key reference (adds author_id + index)
bin/rails generate model Article title:string author:references

# With options
bin/rails generate model Product name:string:index price:decimal{10,2} sku:string!
```

Run the migration after generating:

```bash
bin/rails db:migrate
```

## CRUD Basics

### Create

```ruby
# new + save (returns false on failure, does not raise)
article = Article.new(title: "Hello World", body: "...")
article.save          # => true or false
article.persisted?    # => true if saved

# create (new + save in one step, returns object)
article = Article.create(title: "Hello World", body: "...")
article.persisted?    # => true if saved, false if validations failed

# create! (raises ActiveRecord::RecordInvalid on failure)
article = Article.create!(title: "Hello World", body: "...")
```

### Read

```ruby
# Find by primary key — raises ActiveRecord::RecordNotFound if missing
article = Article.find(1)
articles = Article.find(1, 2, 3)    # returns array

# Find by attribute — returns nil if not found
article = Article.find_by(title: "Hello World")
article = Article.find_by!(title: "Hello World")  # raises if not found

# All records (returns ActiveRecord::Relation — lazy)
articles = Article.all

# Conditions
articles = Article.where(published: true)
articles = Article.where("published_at > ?", 1.week.ago)

# First / last
Article.first       # ORDER BY id ASC LIMIT 1
Article.last        # ORDER BY id DESC LIMIT 1
Article.first(3)    # returns array of 3
```

### Update

```ruby
# update_attributes-style (since Rails 7.1 also: update)
article = Article.find(1)
article.update(title: "Updated", published: true)  # returns true/false
article.update!(title: "Updated")                  # raises on failure

# Mass update via class method
Article.where(published: false).update_all(published: true)

# Direct attribute assignment + save
article.title = "New Title"
article.save

# update_column / update_columns (skip validations and callbacks)
article.update_column(:views_count, 42)
article.update_columns(views_count: 42, updated_at: Time.current)
```

### Destroy

```ruby
article = Article.find(1)
article.destroy     # triggers callbacks, returns the object
article.destroy!    # raises ActiveRecord::RecordNotDestroyed if callbacks halt

# Class-level destroy
Article.find(1).destroy
Article.destroy(1)       # find + destroy
Article.destroy_all      # iterate and destroy each (triggers callbacks)

# delete — skips callbacks, single SQL DELETE
article.delete
Article.delete(1)
Article.where(published: false).delete_all  # single SQL, no callbacks
```

## Model Class Anatomy

```ruby
class Article < ApplicationRecord
  # Associations
  belongs_to :author
  has_many :comments, dependent: :destroy
  has_many :tags, through: :article_tags

  # Validations
  validates :title, presence: true, length: { maximum: 255 }
  validates :body, presence: true
  validates :slug, uniqueness: true

  # Scopes
  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc).limit(10) }

  # Callbacks
  before_validation :normalize_title
  after_create :notify_subscribers

  # Class methods
  def self.search(query)
    where("title ILIKE ?", "%#{query}%")
  end

  # Instance methods
  def publish!
    update!(published: true, published_at: Time.current)
  end

  private

  def normalize_title
    self.title = title.strip.titleize if title.present?
  end
end
```

## Decision Tree: Which Sub-File to Read

```
What do you need to do?
│
├── Create, modify, or roll back database tables?
│   └── migrations.md
│
├── Define relationships between models (belongs_to, has_many, etc.)?
│   └── associations.md
│
├── Add data integrity rules or input validation?
│   └── validations.md
│
├── Query the database — scopes, joins, eager loading, performance?
│   └── queries.md
│
└── Use multiple databases or read replicas?
    └── multi-database.md
```

## Files in This Skill

| File | What It Covers |
|---|---|
| `migrations.md` | Creating, running, rolling back migrations; schema management |
| `associations.md` | belongs_to, has_many, polymorphic, STI, self-referential |
| `validations.md` | Built-in validators, custom validators, callbacks, errors |
| `queries.md` | Finders, scopes, joins, eager loading, N+1, Arel, batching |
| `multi-database.md` | Multiple DBs, read replicas, horizontal sharding |

## Quick Reference

```bash
# Generator shortcuts
bin/rails g model Article                    # alias for generate model
bin/rails g model Article --no-migration     # model only, no migration
bin/rails g model Article --parent=Post      # inherits from Post (STI)

# Console
bin/rails console            # full IRB console with Rails loaded
bin/rails console --sandbox  # all changes rolled back on exit

# Check what columns a model has
bin/rails runner "puts Article.column_names"
```
