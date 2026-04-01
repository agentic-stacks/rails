# Install Ruby

Ruby must be installed before Rails. Use a version manager — never rely on the system Ruby for development.

## Version Requirements

| Rails Version | Minimum Ruby | Recommended |
|---|---|---|
| Rails 7.0 – 7.1 | Ruby 3.0 | Ruby 3.3.x |
| Rails 7.2 | Ruby 3.1 | Ruby 3.3.x |
| Rails 8.x | Ruby 3.2 | Ruby 3.3.x |

**Recommendation: Install Ruby 3.3.6** (latest stable as of early 2025). This satisfies all Rails 7.x and 8.x requirements.

## Choose a Version Manager

```
Which version manager do you want?
├── rbenv  ──► Most common, lightweight, recommended for beginners (Option 1)
├── asdf   ──► Polyglot manager (Node, Python, Ruby in one tool) (Option 2)
├── mise   ──► Fast Rust-based alternative to asdf (Option 3)
└── System Ruby ──► Only for containers/CI, not development (Option 4)
```

---

## Option 1: rbenv (Recommended)

rbenv intercepts Ruby commands via **shims** — lightweight wrapper scripts prepended to your `PATH` in `~/.rbenv/shims/`. When you run `ruby`, the shim checks your project's `.ruby-version` file (or the global setting) and executes the correct Ruby binary.

### macOS (Homebrew)

```bash
# Install rbenv and ruby-build (the plugin that compiles Ruby)
brew install rbenv ruby-build

# Initialize rbenv in your shell (zsh is default on macOS 10.15+)
echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
source ~/.zshrc

# Verify the installation
rbenv doctor   # optional, checks for common problems
```

If you use bash instead of zsh:

```bash
echo 'eval "$(rbenv init - bash)"' >> ~/.bash_profile
source ~/.bash_profile
```

### Linux (Ubuntu / Debian) — git clone

```bash
# Install build dependencies
sudo apt-get update
sudo apt-get install -y git curl libssl-dev libreadline-dev zlib1g-dev \
  autoconf bison build-essential libyaml-dev libncurses5-dev \
  libffi-dev libgdbm-dev

# Clone rbenv
git clone https://github.com/rbenv/rbenv.git ~/.rbenv

# Add rbenv to PATH and initialize
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init - bash)"' >> ~/.bashrc
source ~/.bashrc

# Install ruby-build plugin
git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build
```

### Install Ruby with rbenv

```bash
# List available stable versions
rbenv install --list

# Install a specific version
rbenv install 3.3.6

# Set as the default for all projects
rbenv global 3.3.6

# Optionally set a version for just one project directory
cd /path/to/your/project
rbenv local 3.3.6   # writes .ruby-version file
```

### Verify rbenv Ruby

```bash
ruby -v
# => ruby 3.3.6 (2024-11-05 revision 75015d4c1f) [arm64-darwin24]

which ruby
# => /Users/you/.rbenv/shims/ruby   (must NOT be /usr/bin/ruby)

gem -v
# => 3.5.x
```

---

## Option 2: asdf

asdf manages multiple language runtimes (Ruby, Node, Python, etc.) via a single `.tool-versions` file. Useful if you already use asdf for other tools.

### Install asdf

```bash
# macOS
brew install asdf
echo '. "$(brew --prefix asdf)/libexec/asdf.sh"' >> ~/.zshrc
source ~/.zshrc

# Ubuntu/Debian
sudo apt-get install -y curl git
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.1
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
source ~/.bashrc
```

### Add Ruby Plugin and Install

```bash
# Add the Ruby plugin
asdf plugin add ruby https://github.com/asdf-vm/asdf-ruby.git

# List available versions
asdf list all ruby | tail -20

# Install Ruby 3.3.6
asdf install ruby 3.3.6

# Set global default
asdf global ruby 3.3.6

# Or set per-project (writes .tool-versions)
asdf local ruby 3.3.6
```

The `.tool-versions` file looks like:

```
ruby 3.3.6
nodejs 22.12.0
```

### Verify asdf Ruby

```bash
ruby -v
# => ruby 3.3.6 ...

which ruby
# => /Users/you/.asdf/shims/ruby
```

---

## Option 3: mise

mise is a fast, Rust-based drop-in replacement for asdf. It reads the same `.tool-versions` files and also supports `mise.toml`.

### Install mise

```bash
# macOS
brew install mise
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc
source ~/.zshrc

# Linux
curl https://mise.run | sh
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc
source ~/.bashrc
```

### Install Ruby with mise

```bash
# Install Ruby 3.3.6
mise use --global ruby@3.3.6

# Per-project (writes .tool-versions or mise.toml)
mise use ruby@3.3.6

# List installed versions
mise list
```

---

## Option 4: System Ruby

> **Warning:** Do not use system Ruby for Rails development on macOS or most Linux distros. It is outdated (macOS ships Ruby 2.6), write-protected, and shared across all users. Gem installs require `sudo` and can break system tools.

Acceptable uses:
- Docker containers where you control the base image
- CI environments with a pre-configured image (e.g., `ruby:3.3` on Docker Hub)
- Read-only environments where you only run `bundle exec`

In these cases, verify the Ruby version satisfies your Rails version requirement before proceeding.

---

## Troubleshooting

### `which ruby` shows `/usr/bin/ruby`

rbenv/asdf/mise is not initializing. Check that your shell config file (`.zshrc`, `.bashrc`, `.bash_profile`) has the init line and that you have sourced it:

```bash
source ~/.zshrc   # or ~/.bashrc
which ruby        # should now show a shims path
```

### OpenSSL build failure on Linux

Ruby needs OpenSSL headers. Install them first:

```bash
sudo apt-get install -y libssl-dev
```

If you have OpenSSL 3 and Ruby 3.0 or earlier, you may see compatibility errors. Use Ruby 3.1+ which natively supports OpenSSL 3.

### Apple Silicon (M1/M2/M3) issues

Most Ruby 3.1+ versions build cleanly on Apple Silicon with Homebrew. If you see `configure: error`:

```bash
# Ensure Homebrew ARM paths are set
export PATH="/opt/homebrew/bin:$PATH"
export LDFLAGS="-L/opt/homebrew/opt/openssl@3/lib"
export CPPFLAGS="-I/opt/homebrew/opt/openssl@3/include"
rbenv install 3.3.6
```

### rbenv: command not found after install

The `rbenv` binary is not on your `PATH`. Add it manually:

```bash
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
```

Then add those two lines to your shell config file permanently.

---

## Next Step

Install Rails: see `rails-install.md`.
