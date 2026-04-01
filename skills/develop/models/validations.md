# Validations

Validations ensure data integrity before records are written to the database. They run automatically when you call `save`, `create`, or `update`.

## When Validations Run

These methods trigger validations:

| Method | Raises on failure? |
|---|---|
| `save` | No — returns `false` |
| `save!` | Yes — `ActiveRecord::RecordInvalid` |
| `create` | No — returns object with errors |
| `create!` | Yes — `ActiveRecord::RecordInvalid` |
| `update` | No — returns `false` |
| `update!` | Yes — `ActiveRecord::RecordInvalid` |
| `valid?` | No — returns `true`/`false` |
| `invalid?` | No — returns `true`/`false` |

These methods **skip** validations:

```ruby
record.save(validate: false)
record.update_column(:status, "active")   # skips validations AND callbacks
record.update_columns(status: "active")   # skips validations AND callbacks
Article.where(...).update_all(...)        # skips validations AND callbacks
record.toggle!(:published)                # skips validations
```

Use skipping sparingly. It bypasses your data integrity rules.

## Checking Validity

```ruby
article = Article.new
article.valid?          # => false — triggers validations, returns bool
article.invalid?        # => true

article.errors.full_messages
# => ["Title can't be blank", "Body is too short (minimum is 10 characters)"]

article.errors[:title]
# => ["can't be blank"]

article.errors.details
# => { title: [{ error: :blank }], body: [{ error: :too_short, count: 10 }] }
```

## Built-in Validators

### presence

Validates the attribute is not empty. Calls `blank?` — catches nil, empty string, whitespace-only string.

```ruby
validates :title, presence: true
validates :email, :name, presence: true  # multiple attributes
```

### absence

Validates the attribute IS blank (inverse of presence).

```ruby
validates :legacy_field, absence: true
```

### uniqueness

```ruby
validates :email, uniqueness: true

# Case-insensitive (default in most databases)
validates :username, uniqueness: { case_sensitive: false }

# Scoped uniqueness (unique within a parent)
validates :title, uniqueness: { scope: :author_id }
validates :rank,  uniqueness: { scope: [:category, :year] }
```

Note: uniqueness validation has a race condition under concurrent requests. Add a database-level unique index as the ultimate guard:

```ruby
add_index :users, :email, unique: true
```

### length

```ruby
validates :title,  length: { minimum: 3 }
validates :bio,    length: { maximum: 500 }
validates :pin,    length: { is: 4 }
validates :body,   length: { in: 10..5000 }

# Custom messages per constraint
validates :password, length: {
  minimum: 8,
  too_short: "must be at least %{count} characters"
}
```

### numericality

```ruby
validates :price,       numericality: true
validates :age,         numericality: { only_integer: true }
validates :score,       numericality: { greater_than: 0 }
validates :rating,      numericality: { greater_than_or_equal_to: 1, less_than_or_equal_to: 5 }
validates :quantity,    numericality: { other_than: 0 }
validates :tax_rate,    numericality: { greater_than: 0, less_than: 1 }
```

Options: `greater_than`, `greater_than_or_equal_to`, `equal_to`, `less_than`, `less_than_or_equal_to`, `other_than`, `in`, `odd`, `even`, `only_integer`.

### format

```ruby
validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
validates :slug,  format: { with: /\A[a-z0-9-]+\z/, message: "only lowercase letters, numbers, hyphens" }
validates :code,  format: { without: /\s/, message: "must not contain spaces" }
```

### inclusion / exclusion

```ruby
validates :status, inclusion: { in: %w[draft published archived] }
validates :role,   inclusion: { in: %w[admin editor reader], message: "%{value} is not a valid role" }
validates :subdomain, exclusion: { in: %w[www api admin mail] }
```

### acceptance

For checkbox fields (terms of service):

```ruby
validates :terms_of_service, acceptance: true
validates :eula, acceptance: { accept: ["yes", "YES"] }
```

Does not require a database column (virtual attribute by default).

### confirmation

For two-field confirmation (passwords, emails):

```ruby
validates :email,    confirmation: true
validates :password, confirmation: { case_sensitive: false }
```

Requires a matching `_confirmation` accessor in the form:

```ruby
# form params must include email_confirmation
user.email = "user@example.com"
user.email_confirmation = "user@example.com"
```

### comparison (Rails 7+)

```ruby
validates :start_date, comparison: { less_than: :end_date }
validates :end_date,   comparison: { greater_than: :start_date }
validates :age,        comparison: { greater_than_or_equal_to: 18 }
```

### validates_associated

Validate associated objects when saving the parent:

```ruby
class Author < ApplicationRecord
  has_many :books
  validates_associated :books
end
```

Does not validate if the associated object is `nil`. Does not prevent infinite loops when both sides call `validates_associated`.

## Common Options (apply to all validators)

```ruby
validates :bio,
  length: { maximum: 500 },
  allow_nil: true         # skip validation if value is nil

validates :nickname,
  length: { minimum: 2 },
  allow_blank: true       # skip validation if value is blank

validates :title,
  presence: true,
  message: "is required and cannot be empty"

validates :status,
  inclusion: { in: %w[active inactive] },
  on: :create             # only validate on create (:update also available)
```

## Conditional Validations

### :if and :unless

```ruby
validates :card_number, presence: true, if: :payment_by_card?
validates :paypal_email, presence: true, unless: :payment_by_card?

# Using a proc
validates :discount, numericality: true, if: -> { discount.present? }

# Using a string (avoid — method reference is clearer)
validates :expiry, presence: true, if: "payment_by_card?"

private

def payment_by_card?
  payment_type == "card"
end
```

### with_options

Group validations under a shared condition:

```ruby
with_options if: :premium_user? do
  validates :billing_address, presence: true
  validates :credit_card, presence: true
  validates :subscription_tier, inclusion: { in: %w[monthly annual] }
end
```

### Combining conditions

```ruby
validates :email,
  presence: true,
  if: [:account_active?, -> { requires_email? }]
# All conditions in the array must be true
```

## Validation Contexts

Run validations only on specific contexts:

```ruby
class User < ApplicationRecord
  validates :email, presence: true, on: :create
  validates :age,   numericality: true, on: :update
  validates :token, presence: true, on: :registration  # custom context
end
```

```ruby
user.valid?(:registration)  # runs only validations with on: :registration
user.save(context: :registration)
```

## Custom Validators

### Method-based (simplest)

```ruby
class Article < ApplicationRecord
  validate :title_must_not_contain_profanity
  validate :publication_date_cannot_be_in_past, on: :create

  private

  def title_must_not_contain_profanity
    if title.present? && Profanity.check(title)
      errors.add(:title, "contains inappropriate language")
    end
  end

  def publication_date_cannot_be_in_past
    if publish_on.present? && publish_on < Date.today
      errors.add(:publish_on, "can't be in the past")
    end
  end
end
```

### EachValidator (reusable across models)

```ruby
# app/validators/email_validator.rb
class EmailValidator < ActiveModel::EachValidator
  EMAIL_REGEXP = /\A[^@\s]+@([^@\s]+\.)+[^@\s]+\z/

  def validate_each(record, attribute, value)
    unless value.match?(EMAIL_REGEXP)
      record.errors.add(attribute, options[:message] || "is not a valid email address")
    end
  end
end

# Usage in any model:
class User < ApplicationRecord
  validates :email, presence: true, email: true
end

class Newsletter < ApplicationRecord
  validates :reply_to, email: { message: "must be a deliverable address" }
end
```

### Validator class (for complex, multi-attribute rules)

```ruby
# app/validators/invoice_validator.rb
class InvoiceValidator < ActiveModel::Validator
  def validate(record)
    if record.line_items.empty?
      record.errors.add(:base, "must have at least one line item")
    end

    if record.total < 0
      record.errors.add(:total, "cannot be negative")
    end
  end
end

class Invoice < ApplicationRecord
  validates_with InvoiceValidator
  validates_with InvoiceValidator, on: :create  # with context
end
```

## Working with Errors

```ruby
article = Article.new(title: "")
article.valid?     # => false

# Full messages (human-readable)
article.errors.full_messages
# => ["Title can't be blank", "Body is too short (minimum is 10 characters)"]

# Per-attribute messages
article.errors[:title]
# => ["can't be blank"]

# Error details (machine-readable)
article.errors.details
# => { title: [{ error: :blank }] }

# Add an error manually
article.errors.add(:title, :too_generic, message: "is not specific enough")
article.errors.add(:base, "something went wrong overall")

# Check for specific error
article.errors.where(:title, :blank).any?   # => true

# Count
article.errors.size    # total error count
article.errors.count   # alias

# Clear errors (resets — does not re-validate)
article.errors.clear
```

Displaying errors in a view:

```erb
<% if @article.errors.any? %>
  <div class="alert alert-danger">
    <ul>
      <% @article.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
    </ul>
  </div>
<% end %>
```

## Callbacks

Callbacks hook into the lifecycle of an Active Record object. Use them sparingly — side effects buried in callbacks make code hard to test and debug.

### Callback order

**Creating a record:**
1. `before_validation`
2. `after_validation`
3. `before_save`
4. `before_create`
5. INSERT
6. `after_create`
7. `after_save`
8. `after_commit` / `after_rollback`

**Updating a record:**
1. `before_validation`
2. `after_validation`
3. `before_save`
4. `before_update`
5. UPDATE
6. `after_update`
7. `after_save`
8. `after_commit` / `after_rollback`

**Destroying a record:**
1. `before_destroy`
2. DELETE
3. `after_destroy`
4. `after_commit` / `after_rollback`

### Defining callbacks

```ruby
class User < ApplicationRecord
  before_validation :normalize_email
  before_create     :set_default_role
  after_create      :send_welcome_email
  before_destroy    :check_admin_count, prepend: true

  private

  def normalize_email
    self.email = email.strip.downcase if email.present?
  end

  def set_default_role
    self.role ||= "reader"
  end

  def send_welcome_email
    UserMailer.welcome(self).deliver_later
  end

  def check_admin_count
    if role == "admin" && User.where(role: "admin").count <= 1
      throw :abort  # halts the callback chain and prevents destruction
    end
  end
end
```

Throw `:abort` in any `before_*` callback to halt the chain and prevent the database operation.

### after_commit vs after_save

```ruby
# after_save runs inside the transaction — database may still roll back
after_save :enqueue_job   # risky: job enqueued even if outer transaction rolls back

# after_commit runs only after the transaction commits — use for side effects
after_commit :enqueue_job, on: :create
after_commit :notify_observers, on: [:create, :update]
after_rollback :log_failure
```

Prefer `after_commit` for anything that has external side effects (emails, jobs, webhooks).

### Conditional callbacks

```ruby
before_save :encrypt_ssn, if: :ssn_changed?
after_create :notify_team, unless: :test_environment?
```

### Callback classes (extract for reuse)

```ruby
class AuditCallback
  def before_destroy(record)
    AuditLog.create!(action: "destroy", record_type: record.class.name, record_id: record.id)
  end
end

class Order < ApplicationRecord
  before_destroy AuditCallback.new
end
```

## When Not to Use Callbacks

Callbacks are appropriate for:
- Normalizing data before save (`before_validation`, `before_save`)
- Setting computed defaults on create (`before_create`)
- Cleaning up dependent non-DB resources on destroy

Avoid callbacks for:
- Sending emails or enqueuing jobs (put in the controller or a service object — use `after_commit` at minimum)
- Complex business logic (use service objects/interactors)
- Cross-model side effects (hard to test in isolation)
- Anything that should be explicitly called rather than implicit

```ruby
# Prefer this — explicit, testable
def UsersController#create
  @user = User.new(user_params)
  if @user.save
    UserMailer.welcome(@user).deliver_later
    redirect_to @user
  else
    render :new
  end
end

# Over this — implicit, surprising
class User < ApplicationRecord
  after_create :send_welcome_email  # fires in tests, seeds, imports...
end
```
