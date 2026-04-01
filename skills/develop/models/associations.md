# Associations

Associations declare relationships between Active Record models. Rails uses them to generate SQL joins, maintain foreign keys, and provide convenience methods for navigating related objects.

## Overview of Association Types

| Association | Declares on | Foreign key lives on | Use when |
|---|---|---|---|
| `belongs_to` | child model | same table | this model holds the FK |
| `has_one` | parent model | other table | one associated record |
| `has_many` | parent model | other table | many associated records |
| `has_many :through` | parent model | join table | many-to-many via join model |
| `has_one :through` | parent model | join table | one-to-one via join model |
| `has_and_belongs_to_many` | either | join table (no model) | simple many-to-many |

## belongs_to

The `belongs_to` side always holds the foreign key column.

```ruby
# Migration
class CreateBooks < ActiveRecord::Migration[8.0]
  def change
    create_table :books do |t|
      t.string :title, null: false
      t.references :author, null: false, foreign_key: true
      t.timestamps
    end
  end
end

# Model
class Book < ApplicationRecord
  belongs_to :author
end
```

`belongs_to` is required by default in Rails 5+. To make it optional:

```ruby
belongs_to :editor, optional: true
```

Options:

```ruby
belongs_to :author,
  class_name: "User",           # model class is User, not Author
  foreign_key: "written_by_id", # override default foreign key
  primary_key: "uuid",          # override default primary key
  optional: true,               # do not validate presence
  inverse_of: :books,           # hint for bidirectional caching
  touch: true                   # update author.updated_at on save
```

## has_one

`has_one` declares that the foreign key lives on the other table.

```ruby
class User < ApplicationRecord
  has_one :profile
end

class Profile < ApplicationRecord
  belongs_to :user
  # profiles table has user_id column
end
```

```ruby
user.profile                    # SELECT * FROM profiles WHERE user_id = ?
user.build_profile(bio: "...")  # new, not saved
user.create_profile!(bio: "...") # saved, raises on failure
```

## has_many

```ruby
class Author < ApplicationRecord
  has_many :books
  has_many :published_books, -> { where(published: true) }, class_name: "Book"
end

class Book < ApplicationRecord
  belongs_to :author
end
```

Common options:

```ruby
has_many :books,
  class_name: "Publication",      # model class differs from association name
  foreign_key: "written_by_id",   # non-standard FK column
  primary_key: "uuid",            # non-standard PK
  dependent: :destroy,            # destroy each child when parent is destroyed
  inverse_of: :author             # hint for bidirectional association
```

`dependent` options:

| Option | Behavior |
|---|---|
| `:destroy` | Calls `destroy` on each child (triggers callbacks) |
| `:destroy_async` | Enqueues a job to destroy children (Rails 6.1+) |
| `:delete_all` | Single SQL DELETE, no callbacks |
| `:nullify` | Sets FK to NULL, no callbacks |
| `:restrict_with_exception` | Raises if children exist |
| `:restrict_with_error` | Adds error to parent, does not destroy |

## has_many :through

Use when you need access to the join model's own attributes, or want to query across three levels.

```ruby
# Join model carries its own data
class Appointment < ApplicationRecord
  belongs_to :physician
  belongs_to :patient
  # appointments table: physician_id, patient_id, scheduled_at
end

class Physician < ApplicationRecord
  has_many :appointments
  has_many :patients, through: :appointments
end

class Patient < ApplicationRecord
  has_many :appointments
  has_many :physicians, through: :appointments
end
```

```ruby
physician = Physician.find(1)
physician.patients                       # all patients via appointments
physician.appointments.create!(patient: patient, scheduled_at: 1.day.from_now)
```

Chaining through associations:

```ruby
class Document < ApplicationRecord
  has_many :sections
  has_many :paragraphs, through: :sections  # chain two levels
end
```

## has_one :through

```ruby
class Supplier < ApplicationRecord
  has_one :account
  has_one :account_history, through: :account
end

class Account < ApplicationRecord
  belongs_to :supplier
  has_one :account_history
end

class AccountHistory < ApplicationRecord
  belongs_to :account
end
```

## has_and_belongs_to_many (HABTM)

Use only when the join relationship has no attributes of its own. Prefer `has_many :through` when in doubt.

```ruby
# Migration — join table has no id, no timestamps
class CreateAssembliesParts < ActiveRecord::Migration[8.0]
  def change
    create_join_table :assemblies, :parts do |t|
      t.index :assembly_id
      t.index :part_id
    end
  end
end

class Assembly < ApplicationRecord
  has_and_belongs_to_many :parts
end

class Part < ApplicationRecord
  has_and_belongs_to_many :assemblies
end
```

## Polymorphic Associations

A model can `belongs_to` multiple other models through a single association. Requires two columns: `<name>_id` and `<name>_type`.

```ruby
# Migration
class CreatePictures < ActiveRecord::Migration[8.0]
  def change
    create_table :pictures do |t|
      t.string :name
      t.references :imageable, polymorphic: true, null: false
      # Creates imageable_id (bigint) and imageable_type (string)
      t.timestamps
    end
  end
end

# Polymorphic model
class Picture < ApplicationRecord
  belongs_to :imageable, polymorphic: true
end

# Owners declare the other side
class Employee < ApplicationRecord
  has_many :pictures, as: :imageable
end

class Product < ApplicationRecord
  has_many :pictures, as: :imageable
end
```

```ruby
employee = Employee.find(1)
employee.pictures.create!(name: "headshot.jpg")

picture = Picture.find(1)
picture.imageable   # => Employee or Product object
```

Common polymorphic patterns: comments, attachments, tags, addresses, notifications.

## Single Table Inheritance (STI)

STI stores multiple model subclasses in a single table. A `type` column holds the class name.

```ruby
# Migration
class CreateVehicles < ActiveRecord::Migration[8.0]
  def change
    create_table :vehicles do |t|
      t.string :type, null: false        # required for STI
      t.string :color
      t.integer :doors_count
      t.decimal :payload_capacity
      t.timestamps
    end
  end
end

# Models
class Vehicle < ApplicationRecord
  # base class — shared columns and behaviour
end

class Car < Vehicle
  # SELECT * FROM vehicles WHERE type = 'Car'
end

class Truck < Vehicle
  # SELECT * FROM vehicles WHERE type = 'Truck'
end

class Motorcycle < Vehicle
end
```

```ruby
Car.all        # => only cars
Vehicle.all    # => all vehicles
car = Car.create!(color: "red", doors_count: 4)
car.type       # => "Car"
```

STI works well when subclasses share most columns and behaviour. Avoid it when subclasses diverge significantly — use separate tables with polymorphic associations or delegated types instead.

### Delegated Types (Rails 6.1+)

An alternative to STI that avoids a wide, sparse table:

```ruby
# entries table: entryable_type, entryable_id, account_id, created_at
class Entry < ApplicationRecord
  belongs_to :account
  delegated_type :entryable, types: %w[Message Comment]
end

class Message < ApplicationRecord
  # messages table: subject, body
  has_one :entry, as: :entryable, touch: true
end

class Comment < ApplicationRecord
  # comments table: content
  has_one :entry, as: :entryable, touch: true
end
```

## Self-Referential Associations

Use when a model relates to other instances of itself (tree structures, friend networks, org charts).

```ruby
# employees table has manager_id column
class Employee < ApplicationRecord
  has_many :direct_reports,
    class_name: "Employee",
    foreign_key: "manager_id"

  belongs_to :manager,
    class_name: "Employee",
    optional: true

  # Recursive — all subordinates (using PostgreSQL recursive CTE or acts_as_tree gem)
  def all_subordinates
    direct_reports.flat_map { |r| [r] + r.all_subordinates }
  end
end
```

```ruby
employee.direct_reports       # employees managed by this employee
employee.manager              # this employee's manager
```

Tree gems for deeper hierarchies: `ancestry`, `closure_tree`, `acts_as_tree`.

## Association Scopes

Add default conditions to any association:

```ruby
class Author < ApplicationRecord
  has_many :published_books, -> { where(published: true) }, class_name: "Book"
  has_many :recent_books, -> { order(created_at: :desc).limit(5) }, class_name: "Book"
end
```

## Counter Cache

Cache an association count in a column to avoid `COUNT(*)` queries:

```ruby
class Book < ApplicationRecord
  belongs_to :author, counter_cache: true
  # authors table needs reviews_count column (integer, default: 0)
end
```

Or use a custom column name:

```ruby
belongs_to :author, counter_cache: :books_count
```

Reset a stale counter:

```bash
bin/rails runner "Author.find_each { |a| Author.reset_counters(a.id, :books) }"
```

## touch

Propagate `updated_at` changes up the association tree:

```ruby
class Comment < ApplicationRecord
  belongs_to :article, touch: true
  # Saving a comment updates article.updated_at
end
```

## Association Methods Summary

```ruby
# has_many collection methods
author.books                      # load collection
author.books.build(title: "...")  # new, not saved
author.books.create!(title: "...") # saved, raises on failure
author.books << book              # append (and save join)
author.books.delete(book)         # remove (nullifies FK by default)
author.books.destroy(book)        # remove (runs callbacks)
author.books.clear                # remove all
author.books.count                # SQL COUNT
author.books.size                 # count from cache or SQL
author.books.empty?               # true if none
author.books.include?(book)       # true if in collection
author.books.where(published: true) # further scope

# has_one / belongs_to
user.build_profile(bio: "...")     # new, not saved
user.create_profile!(bio: "...")   # saved, raises on failure
user.profile = profile             # assign (and save)
```
