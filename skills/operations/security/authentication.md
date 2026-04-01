# Authentication

Authentication verifies who the user is. Rails 8 ships a generator for a complete auth system. For simpler cases, `has_secure_password` is enough. Devise is the full-featured option for complex requirements.

---

## Rails 8 Authentication Generator

Rails 8 includes a production-ready generator that creates a full session-based auth system.

```bash
bin/rails generate authentication
```

This creates:

```
app/models/user.rb               # User with has_secure_password
app/models/session.rb            # Session model with token
app/controllers/sessions_controller.rb  # login / logout
app/controllers/passwords_controller.rb # password reset flow
app/mailers/passwords_mailer.rb
app/views/sessions/new.html.erb
db/migrate/*_create_users.rb
db/migrate/*_create_sessions.rb
```

Run the migration:

```bash
bin/rails db:migrate
```

Protect actions with the generated concern:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Authentication  # generated concern

  before_action :require_authentication
end

# Opt out for public actions
class ArticlesController < ApplicationController
  allow_unauthenticated_access only: [:index, :show]
end
```

The generated `Authentication` concern provides `authenticated_user`, `require_authentication`, and `start_new_session_for`.

---

## has_secure_password

Use `has_secure_password` for apps that need basic username/password auth without the full Devise stack.

### Setup

Add bcrypt to Gemfile:

```ruby
# Gemfile
gem "bcrypt", "~> 3.1"
```

Add a `password_digest` column to the users table:

```bash
bin/rails generate migration AddPasswordDigestToUsers password_digest:string
bin/rails db:migrate
```

Enable in the model:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  validates :email, presence: true, uniqueness: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, length: { minimum: 12 }, if: -> { new_record? || password.present? }
end
```

`has_secure_password` adds:
- `password` and `password_confirmation` virtual attributes
- Automatic bcrypt hashing into `password_digest`
- `authenticate(plain_text)` — returns the user or `false`

### Session-based Login

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  skip_before_action :require_login, only: [:new, :create]

  def create
    user = User.find_by(email: params[:email].downcase.strip)
    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      redirect_to root_path, notice: "Signed in."
    else
      flash.now[:alert] = "Invalid email or password."
      render :new, status: :unprocessable_entity
    end
  end

  def destroy
    session.delete(:user_id)
    redirect_to login_path
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :require_login

  private

  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
  helper_method :current_user

  def require_login
    redirect_to login_path unless current_user
  end
end
```

---

## Devise

Use Devise when you need email confirmation, password reset, account locking, OAuth, or multi-tenancy. It covers features that would take weeks to build correctly from scratch.

### Install

```ruby
# Gemfile
gem "devise"
```

```bash
bundle install
bin/rails generate devise:install
```

Follow the printed instructions: set `config.action_mailer.default_url_options` and add a root route.

### Generate a Devise Model

```bash
bin/rails generate devise User
bin/rails db:migrate
```

### Modules

Enable modules in the model:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable,   # hashed password, authenticate
         :registerable,               # sign-up flow
         :recoverable,                # password reset via email
         :rememberable,               # "remember me" cookie
         :validatable,                # email/password validations
         :confirmable,                # email confirmation before sign-in
         :lockable,                   # lock after N failed attempts
         :trackable,                  # sign_in_count, last_sign_in_at
         :timeoutable                 # session expiry after inactivity
end
```

Only enable what you need. Each module adds columns; run a migration when adding a module to an existing model.

### Generated Helpers

```ruby
# Controllers
before_action :authenticate_user!       # redirect to sign-in if not logged in
current_user                            # the signed-in User instance
user_signed_in?                         # boolean

# Routes (added automatically)
new_user_session_path       # /users/sign_in
destroy_user_session_path   # /users/sign_out
new_user_registration_path  # /users/sign_up
```

### Customize Views

```bash
bin/rails generate devise:views
# Generates ERB templates in app/views/devise/
```

### Customize Controllers

```bash
bin/rails generate devise:controllers users
```

Then point routes to the custom controllers:

```ruby
# config/routes.rb
devise_for :users, controllers: {
  sessions: "users/sessions",
  registrations: "users/registrations"
}
```

---

## When to Use Which

| Scenario | Recommendation |
|---|---|
| New Rails 8 app, standard session auth | Rails 8 generator |
| Simple app, no email confirmation needed | `has_secure_password` |
| Need email confirmation, password reset, lockout | Devise |
| Need OAuth / social login | Devise + `omniauth` |
| API-only app | Token auth or JWT (see below) |

---

## Session Management

Always regenerate the session ID after login to prevent session fixation:

```ruby
# has_secure_password / custom auth
reset_session
session[:user_id] = user.id
```

Devise handles this automatically.

Set a reasonable session timeout:

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: "_myapp_session",
  expire_after: 30.minutes,
  secure: Rails.env.production?,
  same_site: :lax
```

---

## API Authentication

### Token Authentication

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_token :api_token  # generates a random token on create
end

# app/controllers/api/application_controller.rb
module Api
  class ApplicationController < ActionController::API
    before_action :authenticate_by_token

    private

    def authenticate_by_token
      token = request.headers["Authorization"]&.sub(/\ABearer /, "")
      @current_user = User.find_by(api_token: token)
      render json: { error: "Unauthorized" }, status: :unauthorized unless @current_user
    end
  end
end
```

### JWT

Prefer signed tokens with short expiry. Use the `jwt` gem:

```ruby
gem "jwt"
```

```ruby
# lib/json_web_token.rb
module JsonWebToken
  SECRET = Rails.application.credentials.secret_key_base
  ALGORITHM = "HS256"
  EXPIRY = 24.hours

  def self.encode(payload)
    payload[:exp] = EXPIRY.from_now.to_i
    JWT.encode(payload, SECRET, ALGORITHM)
  end

  def self.decode(token)
    JWT.decode(token, SECRET, true, algorithm: ALGORITHM).first
  rescue JWT::DecodeError
    nil
  end
end
```

Never store JWTs in localStorage — use HttpOnly cookies or keep expiry short with a refresh token strategy.
