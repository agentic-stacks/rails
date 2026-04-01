# Authorization

Authorization answers "what is this user allowed to do?" after authentication has answered "who are they?". Keep authorization out of models and views — put it in dedicated policy or ability objects.

---

## Pundit

Pundit uses plain Ruby policy classes — one per resource. It is explicit, easy to test, and has no magic.

### Install

```ruby
# Gemfile
gem "pundit"
```

```bash
bundle install
bin/rails generate pundit:install
# Creates app/policies/application_policy.rb
```

Include in ApplicationController:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pundit::Authorization
  after_action :verify_authorized, except: :index
  after_action :verify_policy_scoped, only: :index
end
```

`verify_authorized` raises `Pundit::AuthorizationNotPerformedError` if a controller action returns without calling `authorize`. This prevents accidental unprotected endpoints.

### Write a Policy

```ruby
# app/policies/article_policy.rb
class ArticlePolicy < ApplicationPolicy
  # record is the Article instance; user is current_user

  def index?   = true            # anyone can list
  def show?    = true            # anyone can read

  def create?  = user.present?  # logged-in users only

  def update?
    user.present? && (user.admin? || record.author == user)
  end

  def destroy?
    user.admin? || record.author == user
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      if user&.admin?
        scope.all
      elsif user
        scope.where(published: true).or(scope.where(author: user))
      else
        scope.where(published: true)
      end
    end
  end
end
```

### Authorize in Controllers

```ruby
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  def index
    @articles = policy_scope(Article)  # calls Scope#resolve
  end

  def show
    @article = Article.find(params[:id])
    authorize @article            # calls ArticlePolicy#show?
  end

  def update
    @article = Article.find(params[:id])
    authorize @article            # calls ArticlePolicy#update?
    @article.update!(article_params)
    redirect_to @article
  end

  def destroy
    @article = Article.find(params[:id])
    authorize @article            # calls ArticlePolicy#destroy?
    @article.destroy!
    redirect_to articles_path
  end
end
```

### Handle Authorization Failure

```ruby
# app/controllers/application_controller.rb
rescue_from Pundit::NotAuthorizedError do |e|
  redirect_to root_path, alert: "You are not authorized to perform that action."
end
```

### Policy Helpers in Views

```erb
<% if policy(@article).update? %>
  <%= link_to "Edit", edit_article_path(@article) %>
<% end %>
```

---

## CanCanCan

CanCanCan centralizes all permissions in a single `Ability` class. It is a better fit when permissions are data-driven (stored in the database) or when you need role inheritance.

### Install

```ruby
# Gemfile
gem "cancancan"
```

```bash
bundle install
bin/rails generate cancan:ability
# Creates app/models/ability.rb
```

### Define Abilities

```ruby
# app/models/ability.rb
class Ability
  include CanCan::Ability

  def initialize(user)
    user ||= User.new  # guest user — no id

    if user.admin?
      can :manage, :all  # admin can do anything
    elsif user.persisted?
      can :read, Article, published: true
      can :read, Article, author_id: user.id
      can :create, Article
      can :update, Article, author_id: user.id
      can :destroy, Article, author_id: user.id
      can :manage, Comment, user_id: user.id
    else
      can :read, Article, published: true
    end
  end
end
```

`can :manage, Resource` grants all CRUD actions. Use `cannot` to subtract permissions.

### Authorize in Controllers

```ruby
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  load_and_authorize_resource  # loads @article and calls authorize! automatically

  def index
    # @articles already set by load_and_authorize_resource
  end

  def show
    # @article already loaded and authorized
  end

  def update
    @article.update!(article_params)
    redirect_to @article
  end
end
```

`load_and_authorize_resource` calls `Article.find(params[:id])` for member actions and `Article.accessible_by(current_ability)` for collection actions.

For fine-grained control, authorize explicitly:

```ruby
def publish
  @article = Article.find(params[:id])
  authorize! :publish, @article   # checks can?(:publish, @article)
  @article.publish!
end
```

### Check Permissions in Views

```erb
<% if can? :update, @article %>
  <%= link_to "Edit", edit_article_path(@article) %>
<% end %>
```

---

## When to Use Which

| Scenario | Recommendation |
|---|---|
| Permissions vary per resource, explicit policies preferred | Pundit |
| Permissions stored in DB or role hierarchy | CanCanCan |
| Team prefers single file for all permissions | CanCanCan |
| Team prefers one file per resource | Pundit |
| Need per-object scoping in queries | Either (both support it) |

For new apps without existing conventions, Pundit's explicitness makes authorization failures easier to find and test.

---

## Common Patterns

### Role-Based Access Control

Store roles in an enum or array column:

```ruby
# migration
add_column :users, :role, :string, default: "member", null: false

# model
class User < ApplicationRecord
  ROLES = %w[member editor admin].freeze
  validates :role, inclusion: { in: ROLES }

  def admin?  = role == "admin"
  def editor? = role == "editor"
end
```

```ruby
# Pundit policy with roles
def update?
  user.admin? || user.editor? || record.author == user
end
```

### Resource-Based Access Control (Ownership)

For simple ownership checks, a shared concern keeps policies DRY:

```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
  def owned_by_user?
    record.respond_to?(:user_id) && record.user_id == user&.id
  end
end
```

### Scoping API Responses

Always scope queries to what the current user can see:

```ruby
# Pundit
def index
  @articles = policy_scope(Article)
end

# CanCanCan
def index
  @articles = Article.accessible_by(current_ability)
end
```

Never call `Article.all` in a controller without scoping through the authorization layer.

---

## Testing Policies

### Testing Pundit Policies

Test policy classes directly — no controller setup needed.

```ruby
# test/policies/article_policy_test.rb
require "test_helper"

class ArticlePolicyTest < ActiveSupport::TestCase
  def setup
    @admin   = users(:admin)
    @author  = users(:alice)
    @visitor = users(:bob)
    @article = articles(:published)
    @draft   = Article.create!(title: "Draft", author: @author, published: false)
  end

  test "anyone can read a published article" do
    assert ArticlePolicy.new(nil, @article).show?
  end

  test "author can update their own article" do
    assert ArticlePolicy.new(@author, @draft).update?
  end

  test "other user cannot update someone else's article" do
    refute ArticlePolicy.new(@visitor, @draft).update?
  end

  test "admin can update any article" do
    assert ArticlePolicy.new(@admin, @draft).update?
  end

  test "scope excludes drafts from non-authors" do
    scope = ArticlePolicy::Scope.new(@visitor, Article).resolve
    refute_includes scope, @draft
  end
end
```

### Testing CanCanCan Abilities

```ruby
# test/models/ability_test.rb
require "test_helper"

class AbilityTest < ActiveSupport::TestCase
  test "admin can manage all articles" do
    ability = Ability.new(users(:admin))
    assert ability.can?(:destroy, articles(:any))
  end

  test "member can only update own articles" do
    user    = users(:alice)
    own     = Article.new(author: user)
    others  = Article.new(author: users(:bob))
    ability = Ability.new(user)

    assert ability.can?(:update, own)
    refute ability.can?(:update, others)
  end

  test "guest can only read published articles" do
    ability = Ability.new(nil)
    assert ability.can?(:read, Article.new(published: true))
    refute ability.can?(:read, Article.new(published: false))
  end
end
```
