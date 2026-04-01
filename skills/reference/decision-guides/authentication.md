# Authentication Decision Guide

Choose between the Rails 8 authentication generator, `has_secure_password`, and Devise.

---

## Comparison Table

| Criterion | Rails 8 Generator | has_secure_password | Devise |
|---|---|---|---|
| **Rails version** | 8.0+ | 3.1+ | 3.0+ (gem) |
| **Setup complexity** | Low (generator) | Low (manual) | Medium (generator + config) |
| **Code ownership** | Full (generated code) | Full (you write it) | Partial (gem internals) |
| **Registration flow** | Generated, customisable | Manual | Included |
| **Login / logout** | Generated | Manual | Included |
| **Password recovery** | Generated | Manual | Included |
| **Email confirmation** | Not included | Manual | Via module (`confirmable`) |
| **Remember me** | Generated | Manual | Via module (`rememberable`) |
| **Account locking** | Not included | Manual | Via module (`lockable`) |
| **OAuth / OmniAuth** | Not included | Manual | Via `omniauth` + `devise-omniauth` |
| **Two-factor auth** | Not included | Manual or via gem | Via `devise-two-factor` |
| **Session management** | Generated (single session) | Manual | Included |
| **Customisability** | Very high (you own the code) | Very high | Medium (override controllers/views) |
| **API authentication** | Manual after generation | Manual | Via `devise-jwt` or `devise-token-auth` |
| **Magic links / passwordless** | Manual | Manual | Via `devise-passwordless` |
| **Maintenance burden** | Yours | Yours | Gem team + yours |
| **Upgrade friction** | Low | Low | Medium (Devise upgrades can break things) |
| **Documentation quality** | Good (new, improving) | Excellent (Rails docs) | Excellent |

---

## Option Details

### Rails 8 Authentication Generator

Rails 8 ships a built-in authentication generator that scaffolds a complete, production-ready session-based authentication system.

```bash
bin/rails generate authentication
bin/rails db:migrate
```

Generated files:
- `app/models/user.rb` — with `has_secure_password`, email uniqueness
- `app/models/session.rb` — tracks login sessions per device
- `app/controllers/sessions_controller.rb` — login/logout
- `app/controllers/passwords_controller.rb` — password reset via email token
- `app/controllers/concerns/authentication.rb` — `require_authentication`, `Current.user`
- `app/mailers/passwords_mailer.rb` — reset email
- Views for login, password reset

**Strengths:**
- Minimal dependencies — no gem to maintain or upgrade
- Generated code is readable and fully owned by your team
- Multi-session support out of the box (one row per logged-in device)
- Matches the Rails 8 idiom; well-integrated with Solid Queue for async email

**Weaknesses:**
- Does not include: email confirmation, OAuth, 2FA, account locking, remember-me
- Everything beyond basic login/reset must be built manually
- Young — patterns and community conventions still forming

**When to avoid:**
- Apps that need email confirmation, OAuth, or 2FA without building those features from scratch
- Teams who want a proven, feature-complete solution with minimal custom code

---

### has_secure_password

A Rails module (`bcrypt`-backed) that adds password hashing, validation, and `authenticate` to any model.

```ruby
# Gemfile
gem "bcrypt"
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  validates :email, presence: true, uniqueness: true
end
```

**Strengths:**
- Minimal — just hashing and validation
- No opinions about sessions, controllers, or views
- Appropriate base for custom authentication built exactly to spec
- Available since Rails 3.1; well-understood

**Weaknesses:**
- Provides only the model layer — sessions, controllers, mailers, views must all be written manually
- More code to write and test than using the Rails 8 generator
- No community patterns for "how to add X" — you design everything

**When to use:**
- API-only apps where you control the authentication flow completely
- Apps with very non-standard authentication requirements (e.g., magic links only, biometric fallback)
- As the model foundation under a custom authentication system

**When to avoid:**
- Full-stack web apps where writing all the session/controller/view code is pure boilerplate
- Teams without strong authentication domain knowledge

---

### Devise

The most widely used authentication gem for Rails. A modular engine that provides full authentication out of the box.

```ruby
# Gemfile
gem "devise"
```

```bash
bin/rails generate devise:install
bin/rails generate devise User
bin/rails db:migrate
```

Modules (opt-in): `database_authenticatable`, `registerable`, `recoverable`, `rememberable`, `validatable`, `confirmable`, `lockable`, `timeoutable`, `trackable`, `omniauthable`.

**Strengths:**
- Feature-complete: registration, login, password reset, email confirmation, account locking, OAuth all included or trivially added
- Huge ecosystem: `devise-jwt`, `devise-two-factor`, `devise-passwordless`, `devise-authy`, and more
- Well-documented with a large community
- Battle-tested in millions of Rails apps

**Weaknesses:**
- Gem internals are complex; debugging non-standard flows requires understanding Warden and Devise internals
- Customising views or controllers requires ejecting them with generators, after which upgrades may conflict
- Upgrades can be friction-heavy (`devise >= 4.9` required for Rails 7.1+ due to token changes)
- Heavier than needed for simple apps
- API authentication requires additional gems; Devise was designed for session-based web apps

**When to avoid:**
- Apps that need deep customisation of the auth flow — you spend more time fighting Devise than building
- API-only apps where session-based auth is irrelevant
- Teams that prefer to own all authentication code

---

## Recommendations by Use Case

| Use Case | Recommended | Rationale |
|---|---|---|
| New Rails 8 app, basic email + password login | Rails 8 generator | Zero dependencies, full code ownership, production-ready |
| New Rails 8 app, needs email confirmation + OAuth | Devise | Provides both out of the box; worth the complexity |
| API-only app with JWT | has_secure_password + custom | No session layer needed; token auth is bespoke by nature |
| App needing 2FA | Devise + `devise-two-factor` | Fastest path to working 2FA |
| App with multiple user types (admin, member, guest) | Rails 8 generator or Devise | Generator gives flexibility; Devise with STI for complex cases |
| Existing app using Devise | Keep Devise | Migration cost is not justified without a specific problem |
| Non-standard auth (magic links, SSO) | has_secure_password + custom | Devise fights you; owning the code is easier |

---

## Adding Features to the Rails 8 Generator

The generator is a starting point. Common additions:

### Email Confirmation

Add a `confirmed_at` column and verify it in the `Authentication` concern:
```ruby
# db/migrate/..._add_confirmed_at_to_users.rb
add_column :users, :confirmed_at, :datetime
```
```ruby
# app/controllers/concerns/authentication.rb
def require_authentication
  resume_session || request_authentication
  # Add: redirect_to unconfirmed_path unless Current.user.confirmed?
end
```

### Remember Me

Add a persistent cookie that creates a new session:
```ruby
# Set cookie duration in sessions_controller.rb
cookies.permanent.signed[:session_id] = session.id if params[:remember_me]
```

### OAuth

Use OmniAuth directly without Devise:
```ruby
# Gemfile
gem "omniauth"
gem "omniauth-github"
gem "omniauth-rails_csrf_protection"
```
Add a `provider`/`uid` column to `User` and handle the OmniAuth callback in a dedicated controller.
