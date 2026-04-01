# Bootstrap: Set Up a Rails Development Environment

This skill gets you from zero to a working Rails development environment. Follow the decision tree below to find your starting point, then work through each step in order.

## Decision Tree

```
Do you have Ruby 3.1 or later installed?
├── No  ──► ruby-install.md
└── Yes
    └── Do you have Rails installed?
        ├── No  ──► rails-install.md
        └── Yes
            └── Does your Rails app use jsbundling-rails or cssbundling-rails?
                ├── Yes ──► node-install.md
                └── No (importmap + Propshaft — Rails 8 default)
                    └── Do you want editor support (LSP, autocomplete)?
                        ├── Yes ──► editor-setup.md
                        └── No  ──► Done. See "Next Steps" below.
```

Check what you have installed:

```bash
ruby -v        # Need 3.1.x or higher for Rails 7, 3.2.x or higher for Rails 8
rails -v       # Should show Rails 7.x or 8.x
node -v        # Only needed for jsbundling/cssbundling
```

## Quick Start: macOS

```bash
# 1. Install rbenv via Homebrew
brew install rbenv ruby-build

# 2. Add rbenv to your shell
echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
source ~/.zshrc

# 3. Install Ruby 3.3.x (latest stable)
rbenv install 3.3.6
rbenv global 3.3.6
ruby -v   # => ruby 3.3.6 ...

# 4. Install Rails
gem install rails
rails -v  # => Rails 8.x.x

# 5. (Optional) Install Node if using jsbundling/cssbundling
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
source ~/.zshrc
nvm install 22
nvm alias default 22
node -v   # => v22.x.x
```

## Quick Start: Ubuntu / Debian

```bash
# 1. Install build dependencies
sudo apt-get update
sudo apt-get install -y git curl libssl-dev libreadline-dev zlib1g-dev \
  autoconf bison build-essential libyaml-dev libreadline-dev libncurses5-dev \
  libffi-dev libgdbm-dev

# 2. Install rbenv via git
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init - bash)"' >> ~/.bashrc
source ~/.bashrc

# 3. Install ruby-build plugin
git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build

# 4. Install Ruby 3.3.x
rbenv install 3.3.6
rbenv global 3.3.6
ruby -v   # => ruby 3.3.6 ...

# 5. Install Rails
gem install rails
rails -v  # => Rails 8.x.x

# 6. (Optional) Install Node if using jsbundling/cssbundling
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
source ~/.bashrc
nvm install 22
nvm alias default 22
node -v   # => v22.x.x
```

## Files in This Skill

| File | What It Covers |
|---|---|
| `ruby-install.md` | rbenv, asdf, mise, system Ruby — choose your version manager |
| `rails-install.md` | Installing Rails and Bundler, verifying versions |
| `node-install.md` | Node.js and Yarn — only needed for jsbundling/cssbundling |
| `editor-setup.md` | VS Code, RubyMine, Vim/Neovim, `.editorconfig` |

## Next Steps

Once your environment is ready:

1. **Understand Rails conventions** — `skills/foundation/concepts/`
2. **Create your first app** — `skills/operations/setup/new-app.md`
3. **Configure environments and credentials** — `skills/foundation/configuration/`
