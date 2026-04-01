# Mailer

Action Mailer is Rails' email framework. Mailers inherit from `ActionMailer::Base`, work like controllers, and use ERB templates. Deliver asynchronously via Active Job with `deliver_later`.

## Generate a Mailer

```bash
bin/rails generate mailer UserMailer welcome password_reset
# Creates:
#   app/mailers/user_mailer.rb
#   app/views/user_mailer/welcome.html.erb
#   app/views/user_mailer/welcome.text.erb
#   app/views/user_mailer/password_reset.html.erb
#   app/views/user_mailer/password_reset.text.erb
#   test/mailers/user_mailer_test.rb
#   test/mailers/previews/user_mailer_preview.rb
```

## Mailer Anatomy

```ruby
# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: "notifications@example.com"
  layout "mailer"
end
```

```ruby
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  def welcome(user)
    @user = user
    @login_url = login_url

    mail(
      to: @user.email,
      subject: "Welcome to MyApp"
    )
  end

  def password_reset(user, token)
    @user = user
    @reset_url = edit_password_reset_url(token)

    mail(
      to: @user.email,
      subject: "Reset your password"
    )
  end
end
```

Instance variables assigned in the method are available in both templates.

## Templates

Always provide both an HTML and a plain-text version:

```erb
<%# app/views/user_mailer/welcome.html.erb %>
<!DOCTYPE html>
<html>
  <body>
    <h1>Welcome, <%= @user.first_name %>!</h1>
    <p>
      Get started by <%= link_to "logging in", @login_url %>.
    </p>
  </body>
</html>
```

```erb
<%# app/views/user_mailer/welcome.text.erb %>
Welcome, <%= @user.first_name %>!

Log in at: <%= @login_url %>
```

Rails automatically creates a `multipart/alternative` message containing both parts.

## Sending Mail

```ruby
# Asynchronous — enqueues an Active Job (recommended for production)
UserMailer.welcome(@user).deliver_later

# Synchronous — blocks until the SMTP call completes
UserMailer.welcome(@user).deliver_now

# Deliver later with a delay
UserMailer.welcome(@user).deliver_later(wait: 1.hour)
UserMailer.welcome(@user).deliver_later(wait_until: Date.tomorrow.noon)
```

## Parameterized Mailers

Pass parameters as a hash instead of positional arguments. Useful for shared before-actions and when the mailer is built in multiple steps.

```ruby
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  before_action { @user = params[:user] }

  def welcome
    mail(to: @user.email, subject: "Welcome to MyApp")
  end

  def password_reset
    @token = params[:token]
    @reset_url = edit_password_reset_url(@token)
    mail(to: @user.email, subject: "Reset your password")
  end
end
```

Enqueue with a params hash:

```ruby
UserMailer.with(user: @user).welcome.deliver_later
UserMailer.with(user: @user, token: token).password_reset.deliver_later
```

## Attachments

```ruby
def report(user, report_path)
  @user = user

  attachments["monthly_report.pdf"] = File.read(report_path)

  # Inline attachment (e.g. embedded image)
  attachments.inline["logo.png"] = File.read(Rails.root.join("app/assets/images/logo.png"))

  mail(to: @user.email, subject: "Your monthly report")
end
```

Reference an inline attachment in the template:

```erb
<%= image_tag attachments["logo.png"].url %>
```

## Previews

Previews let you view emails in the browser without sending them.

```ruby
# test/mailers/previews/user_mailer_preview.rb
class UserMailerPreview < ActionMailer::Preview
  def welcome
    UserMailer.welcome(User.first)
  end

  def password_reset
    UserMailer.with(user: User.first, token: "fake-token-123").password_reset
  end
end
```

Browse at `http://localhost:3000/rails/mailers/user_mailer`. Available in development only.

## Configuration

### Production (SMTP)

```ruby
# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address:              "smtp.sendgrid.net",
  port:                 587,
  domain:               "example.com",
  user_name:            Rails.application.credentials.sendgrid_username,
  password:             Rails.application.credentials.sendgrid_password,
  authentication:       :plain,
  enable_starttls_auto: true
}
config.action_mailer.default_url_options = { host: "example.com", protocol: "https" }
```

### Development (letter_opener)

View sent emails in the browser instead of delivering them:

```ruby
# Gemfile
gem "letter_opener", group: :development
```

```ruby
# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.perform_deliveries = true
config.action_mailer.default_url_options = { host: "localhost", port: 3000 }
```

### Test environment

```ruby
# config/environments/test.rb
config.action_mailer.delivery_method = :test
config.action_mailer.default_url_options = { host: "example.com" }
```

Delivered mail accumulates in `ActionMailer::Base.deliveries`:

```ruby
ActionMailer::Base.deliveries.last
ActionMailer::Base.deliveries.clear
```

## Interceptors

Intercept and modify messages before delivery — useful for redirecting all mail to a catch-all address in staging:

```ruby
# config/initializers/mailer_interceptor.rb
class StagingInterceptor
  def self.delivering_email(message)
    message.subject = "[STAGING] #{message.subject}"
    message.to      = ["staging-inbox@example.com"]
  end
end

if Rails.env.staging?
  ActionMailer::Base.register_interceptor(StagingInterceptor)
end
```

## Observer

Execute code after a message is sent (logging, analytics):

```ruby
class MailDeliveryObserver
  def self.delivered_email(message)
    Rails.logger.info "Delivered #{message.subject} to #{message.to.join(", ")}"
  end
end

ActionMailer::Base.register_observer(MailDeliveryObserver)
```

## Testing

```ruby
# test/mailers/user_mailer_test.rb
require "test_helper"

class UserMailerTest < ActionMailer::TestCase
  test "welcome email" do
    user = users(:one)
    email = UserMailer.welcome(user)

    assert_emails 1 do
      email.deliver_now
    end

    assert_equal ["notifications@example.com"], email.from
    assert_equal [user.email],                  email.to
    assert_equal "Welcome to MyApp",            email.subject
    assert_includes email.body.to_s,            user.first_name
  end
end
```

Assert that an email is enqueued (when using `deliver_later`):

```ruby
assert_enqueued_email_with UserMailer, :welcome, args: [user] do
  UserMailer.welcome(user).deliver_later
end
```
