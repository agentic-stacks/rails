# Install Rails

Install Rails after Ruby is in place. Rails is a RubyGem and is installed with the `gem` command.

## Prerequisites

Verify Ruby is ready before continuing:

```bash
ruby -v
# => ruby 3.3.6 ...

which ruby
# Must NOT be /usr/bin/ruby — should be a shims path from rbenv/asdf/mise

gem -v
# => 3.5.x
```

If `which ruby` shows `/usr/bin/ruby`, go back to `ruby-install.md` and configure your version manager.

## Install Rails

### Latest stable (recommended)

```bash
gem install rails
```

This installs the most recent stable release. As of early 2025 this is Rails 8.x.

### Specific minor version (Rails 7.2 series)

```bash
gem install rails -v '~> 7.2'
```

The `~>` pessimistic operator installs the latest patch within the `7.2.x` range.

### Exact version (for reproducibility)

```bash
gem install rails -v '8.0.1'
```

Use this when a project's `Gemfile` pins an exact version and you need to match it locally.

## Verify the Installation

```bash
rails -v
# => Rails 8.0.1

rails --help
# Shows all available rails commands
```

If `rails` is not found after installation, rehash your shims (rbenv only):

```bash
rbenv rehash
```

## Bundler

Bundler is the dependency manager for Ruby projects. It comes bundled with Ruby 2.6 and later — no separate installation needed.

```bash
bundler -v
# => Bundler version 2.5.x

# Also accessible as:
bundle -v
```

If Bundler is missing or outdated:

```bash
gem install bundler
```

## Version Management Per Project

Each Rails project specifies its Rails version in the `Gemfile`:

```ruby
# Gemfile
source "https://rubygems.org"
gem "rails", "~> 8.0"
```

When you run `bundle install`, Bundler resolves and installs the exact versions listed in `Gemfile.lock`. This ensures every developer and CI environment uses identical gem versions.

```bash
cd my-rails-app
bundle install   # installs gems from Gemfile.lock
bin/rails -v     # uses the project-pinned Rails version
```

Always use `bin/rails` inside an application directory to ensure you run the correct version for that project, not a globally-installed Rails.

## Common Issues

### Permission denied when running `gem install`

You are using the system Ruby (owned by root). Switch to your version manager's Ruby:

```bash
which ruby   # check — should NOT be /usr/bin/ruby
```

Never run `sudo gem install` for development. Fix your version manager instead.

### Native extension failures

Some gems compile C extensions. If they fail:

```bash
# macOS — install Xcode Command Line Tools
xcode-select --install

# Ubuntu/Debian — install build tools
sudo apt-get install -y build-essential
```

### `rails` command not found after `gem install rails`

For rbenv, run:

```bash
rbenv rehash
```

For asdf, shims are rebuilt automatically but you can force it:

```bash
asdf reshim ruby
```

### Two Rails versions installed globally

Check what is installed:

```bash
gem list rails
```

You can have multiple versions installed. `gem install rails` installs alongside existing versions. The project's `Gemfile.lock` determines which version `bundle exec rails` uses inside a project directory.

---

## Next Step

- If your Rails app uses `jsbundling-rails` or `cssbundling-rails`: see `node-install.md`.
- Otherwise (importmap + Propshaft, the Rails 8 default): see `editor-setup.md`, then `skills/operations/setup/new-app.md` to create your first app.
