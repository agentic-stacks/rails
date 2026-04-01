# Generators

Rails generators create files from templates so you write less boilerplate. All generators live under `bin/rails generate` (alias `bin/rails g`).

## List Available Generators

```bash
bin/rails generate          # list all generators
bin/rails generate --help   # same
bin/rails generate scaffold --help  # help for a specific generator
```

## Scaffold

Scaffold generates a complete CRUD resource: model, migration, controller, views, routes entry, and tests.

```bash
bin/rails generate scaffold Article title:string body:text status:integer
bin/rails db:migrate
```

Files created:

```
db/migrate/20250101000000_create_articles.rb
app/models/article.rb
app/controllers/articles_controller.rb
app/views/articles/
  _article.html.erb
  _form.html.erb
  index.html.erb
  show.html.erb
  new.html.erb
  edit.html.erb
test/models/article_test.rb
test/controllers/articles_controller_test.rb
test/fixtures/articles.yml
```

The route `resources :articles` is added to `config/routes.rb`.

### Scaffold for API-Only Apps

```bash
bin/rails generate scaffold Article title:string body:text --api
```

Generates controller with JSON responses only — no view files.

## Model Generator

```bash
bin/rails generate model Article title:string body:text published:boolean views_count:integer
```

Creates `app/models/article.rb` and `db/migrate/..._create_articles.rb`. Does not create a controller or views.

Always run the migration after generating:

```bash
bin/rails db:migrate
```

## Controller Generator

```bash
bin/rails generate controller Articles index show new create
```

Creates `app/controllers/articles_controller.rb` with empty action methods, plus a view file for each action listed and a helper.

For a namespaced controller:

```bash
bin/rails generate controller Admin::Articles index
# Creates app/controllers/admin/articles_controller.rb
# Creates app/views/admin/articles/index.html.erb
```

## Migration Generator

Generate a standalone migration without a model:

```bash
# Create a new table
bin/rails generate migration CreateProducts name:string price:decimal

# Add a column to an existing table
bin/rails generate migration AddPublishedToArticles published:boolean

# Add an indexed column
bin/rails generate migration AddSlugToArticles slug:string:index

# Remove a column
bin/rails generate migration RemoveBodyFromArticles body:text

# Add a reference (foreign key + index)
bin/rails generate migration AddAuthorToArticles author:references
```

Rails infers intent from the migration name when it matches these patterns:

| Name pattern | What Rails generates |
|---|---|
| `CreateXxx col:type` | `create_table :xxx` block |
| `AddXxxToYyy col:type` | `add_column :yyy, :xxx, :type` |
| `RemoveXxxFromYyy col:type` | `remove_column :yyy, :xxx, :type` |
| `AddIndexToXxx col` | `add_index :xxx, :col` |

Always review the generated migration before running it.

## Field Types

| Type | SQL | Notes |
|---|---|---|
| `string` | VARCHAR(255) | Short text; use `{n}` to set limit |
| `text` | TEXT | Long text; no length limit |
| `integer` | INTEGER | Whole numbers |
| `bigint` | BIGINT | Large integers; Rails 8 primary key default |
| `float` | FLOAT | Approximate decimal |
| `decimal` | DECIMAL | Exact decimal; use `{precision,scale}` |
| `boolean` | BOOLEAN | true/false |
| `date` | DATE | Date only |
| `datetime` | DATETIME / TIMESTAMP | Date + time |
| `time` | TIME | Time only |
| `json` | JSON | JSON column (PostgreSQL, MySQL 5.7+) |
| `jsonb` | JSONB | Binary JSON (PostgreSQL only) |
| `references` / `belongs_to` | BIGINT + INDEX | Foreign key column + index |
| `binary` | BLOB | Binary data |

## Field Modifiers

Append modifiers to a field with `:` or `{}`:

```bash
# Limit string length to 40 characters
bin/rails generate model User username:string{40}

# Decimal with precision 10, scale 2 (e.g. 12345678.99)
bin/rails generate model Product price:decimal{10,2}

# Add an index on the column
bin/rails generate model User email:string:index

# Add a unique index on the column
bin/rails generate model User email:string:uniq

# NOT NULL constraint (! suffix — Rails 7.1+)
bin/rails generate model User email:string!

# Default value
bin/rails generate migration AddStatusToArticles status:integer

# Multiple modifiers
bin/rails generate model User email:string{255}:uniq
```

## Mailer Generator

```bash
bin/rails generate mailer UserMailer welcome_email password_reset
```

Creates:
- `app/mailers/user_mailer.rb` with `welcome_email` and `password_reset` methods
- `app/views/user_mailer/welcome_email.html.erb`
- `app/views/user_mailer/welcome_email.text.erb`
- `app/views/user_mailer/password_reset.html.erb`
- `app/views/user_mailer/password_reset.text.erb`
- `test/mailers/user_mailer_test.rb`

## Job Generator

```bash
bin/rails generate job ProcessPayment
```

Creates `app/jobs/process_payment_job.rb` inheriting from `ApplicationJob`.

With a specific queue:

```bash
bin/rails generate job ProcessPayment --queue=critical
```

## Channel Generator (Action Cable)

```bash
bin/rails generate channel Notifications
```

Creates:
- `app/channels/notifications_channel.rb`
- `app/javascript/channels/notifications_channel.js`

## Resource Generator

Generates a model + migration + controller with empty actions + routes — but no views (less than scaffold, more than model alone):

```bash
bin/rails generate resource Article title:string body:text
```

Useful as a starting point when you want to build actions manually.

## Destroying Generated Files

Undo a generator by running `destroy` with the same arguments:

```bash
# Undo a scaffold
bin/rails destroy scaffold Article

# Undo a model
bin/rails destroy model Article

# Undo a controller
bin/rails destroy controller Articles

# Undo a migration (removes the file; does NOT roll back if already migrated)
bin/rails destroy migration CreateArticles
```

If you have already run `db:migrate`, roll back first:

```bash
bin/rails db:rollback
bin/rails destroy model Article
```

## Dry Run

Preview what a generator will create without writing any files:

```bash
bin/rails generate scaffold Article title:string --pretend
```

## Customize Generator Defaults

Set per-project defaults in `config/application.rb`:

```ruby
# config/application.rb
module Myapp
  class Application < Rails::Application
    config.generators do |g|
      g.test_framework :rspec           # use RSpec instead of Minitest
      g.fixture_replacement :factory_bot, dir: "spec/factories"
      g.stylesheets false               # skip stylesheet files
      g.helper false                    # skip helper files
      g.assets false                    # skip asset files
      g.view_specs false                # skip view spec files
      g.routing_specs false             # skip routing spec files
    end
  end
end
```

After setting `g.test_framework :rspec`, generators create `spec/` files instead of `test/` files.

## Common Workflows

```bash
# Add a new resource end-to-end
bin/rails generate scaffold Post title:string body:text author:references
bin/rails db:migrate

# Add a column to an existing table
bin/rails generate migration AddPublishedAtToPosts published_at:datetime:index
bin/rails db:migrate

# Check migration status
bin/rails db:migrate:status

# Roll back the last migration
bin/rails db:rollback

# Roll back N migrations
bin/rails db:rollback STEP=3
```

## Next Steps

- Manage gem dependencies: `dependencies.md`
- Write model validations and associations: `skills/develop/models/`
- Write controller actions: `skills/develop/controllers/`
