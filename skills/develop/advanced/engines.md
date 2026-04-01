# Rails Engines

A Rails engine is a miniature Rails application packaged as a Ruby gem. It can have its own models, controllers, views, routes, migrations, and assets. The host application mounts the engine and gets its functionality without duplicating code.

Well-known examples: Devise (authentication), Sidekiq Web (job dashboard), Active Admin (admin interface), Spree (e-commerce).

## Mountable vs Full Engine

| Type | Namespace | Use case |
|---|---|---|
| Mountable | Isolated — own namespace | Reusable features shared across apps (preferred) |
| Full | Shares host namespace | Plugins that extend the host directly (rare) |

Use a mountable engine in almost every case. It avoids constant naming collisions with the host application and makes the engine's scope explicit.

## Generate a Mountable Engine

```bash
# Generate a new mountable engine
bin/rails plugin new my_engine --mountable

# What gets created:
# my_engine/
# ├── app/
# │   ├── controllers/my_engine/application_controller.rb
# │   ├── helpers/my_engine/application_helper.rb
# │   ├── jobs/my_engine/application_job.rb
# │   ├── mailers/my_engine/application_mailer.rb
# │   ├── models/my_engine/application_record.rb
# │   └── views/layouts/my_engine/application.html.erb
# ├── config/
# │   └── routes.rb
# ├── lib/
# │   ├── my_engine/
# │   │   ├── engine.rb
# │   │   └── version.rb
# │   └── my_engine.rb
# ├── my_engine.gemspec
# └── test/
#     └── dummy/          ← minimal Rails app for testing
```

## Engine Class

The engine class wires the gem into the host Rails application:

```ruby
# lib/my_engine/engine.rb
module MyEngine
  class Engine < ::Rails::Engine
    isolate_namespace MyEngine
  end
end
```

`isolate_namespace MyEngine` tells Rails to scope all routes, controllers, and helpers inside the `MyEngine` module. Without it, the engine's routes and helpers bleed into the host.

## Directory Structure (Inside the Engine)

```
my_engine/
├── app/
│   ├── controllers/my_engine/
│   │   ├── application_controller.rb
│   │   └── widgets_controller.rb
│   ├── models/my_engine/
│   │   ├── application_record.rb
│   │   └── widget.rb
│   └── views/my_engine/widgets/
│       ├── index.html.erb
│       └── show.html.erb
├── config/
│   └── routes.rb
├── db/
│   └── migrate/
│       └── 20240101000000_create_my_engine_widgets.rb
└── lib/
    └── my_engine/
        └── engine.rb
```

Engine routes are defined inside `config/routes.rb` relative to the engine's mount point:

```ruby
# my_engine/config/routes.rb
MyEngine::Engine.routes.draw do
  resources :widgets
end
```

## Mount the Engine in the Host App

Add the gem to the host application's `Gemfile`:

```ruby
# Gemfile (host app)
gem "my_engine", path: "../my_engine"        # local development
gem "my_engine", git: "https://github.com/org/my_engine"  # from git
gem "my_engine", "~> 1.0"                   # from RubyGems
```

Mount the engine in the host's routes file:

```ruby
# config/routes.rb (host app)
Rails.application.routes.draw do
  mount MyEngine::Engine, at: "/my_engine"
end
```

The engine is now available at `/my_engine`. Its routes are prefixed by that path.

Generate route helpers that reference engine routes from the host:

```ruby
# In a host view or controller, use the engine's route helper via its proxy:
my_engine.widgets_path        # => "/my_engine/widgets"
my_engine.widget_path(widget) # => "/my_engine/widgets/42"

# From inside the engine, use normal route helpers:
widgets_path
```

## Run Migrations

The engine ships its own migrations. Copy them into the host before running:

```bash
# Copy engine migrations to the host app's db/migrate/
bin/rails my_engine:install:migrations

# Then run them
bin/rails db:migrate
```

Migration timestamps are adjusted on copy to avoid conflicts. Re-run `install:migrations` after upgrading the engine gem to pick up new migrations.

## Engine ApplicationRecord

Isolate the engine's models from the host's database connection handling:

```ruby
# app/models/my_engine/application_record.rb
module MyEngine
  class ApplicationRecord < ActiveRecord::Base
    self.abstract_class = true
  end
end
```

Engine models inherit from this, not from `::ApplicationRecord`:

```ruby
# app/models/my_engine/widget.rb
module MyEngine
  class Widget < ApplicationRecord
    validates :name, presence: true
  end
end
```

## Share a Model with the Host App

If the engine needs to reference a host model (e.g., `User`), accept it via configuration rather than hard-coding it:

```ruby
# lib/my_engine.rb
module MyEngine
  mattr_accessor :user_class, default: "User"
end

# Usage inside the engine:
MyEngine.user_class.constantize.find(id)
```

Configure in the host:

```ruby
# config/initializers/my_engine.rb (host app)
MyEngine.user_class = "Account"
```

## Testing with the Dummy App

The generator creates `test/dummy/` — a minimal Rails application that hosts the engine during tests. Run tests from inside the engine directory:

```bash
cd my_engine
bin/rails test
```

Write integration tests that mount the engine against the dummy app:

```ruby
# test/integration/widgets_test.rb
require "test_helper"

class WidgetsTest < ActionDispatch::IntegrationTest
  include Engine.routes.url_helpers

  test "lists widgets" do
    get widgets_path
    assert_response :success
  end
end
```

## Extract an Existing Feature to an Engine

Follow this sequence to avoid breaking the host while extracting:

1. Generate the engine gem alongside the host app
2. Move files directory-by-directory (models, then controllers, then views)
3. Add `gem "my_engine", path: "../my_engine"` to `Gemfile`
4. Run the test suite after each move
5. Replace host references with engine-namespaced equivalents
6. Delete the originals from the host once tests pass
7. Publish the gem when extraction is complete and stable

## Publish to RubyGems

Update `my_engine.gemspec` with accurate metadata, then:

```bash
cd my_engine
gem build my_engine.gemspec
gem push my_engine-1.0.0.gem
```

Tag the release in git and update `CHANGELOG.md` before pushing.
