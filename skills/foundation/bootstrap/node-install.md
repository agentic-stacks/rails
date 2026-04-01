# Install Node.js and Yarn

Node.js is **not required** for most Rails 8 applications. Read the decision tree below before installing anything.

## Do You Need Node?

```
Which asset pipeline / JS approach does your app use?
│
├── importmap-rails + Propshaft (Rails 8 default, new apps)
│   └── NO — Node is not needed. Skip this file.
│
├── jsbundling-rails (esbuild, rollup, webpack)
│   └── YES — Node is required. Follow this guide.
│
└── cssbundling-rails (Tailwind CLI, PostCSS, Sass)
    └── YES — Node is required. Follow this guide.
```

If you are starting a new app and have not chosen yet:

- **Do not need a complex JS build step?** Use `rails new myapp` (default) — importmap + Propshaft, no Node required.
- **Need esbuild/webpack/Tailwind?** Use `rails new myapp --javascript esbuild` or `--css tailwind` — Node required.

See `skills/develop/assets/` for a full comparison of asset pipeline options.

## Install Node via nvm

nvm (Node Version Manager) lets you install and switch between Node versions without touching system paths.

### Install nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
```

This script clones nvm to `~/.nvm` and adds the following to your shell config (`.bashrc`, `.zshrc`, or `.profile`):

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

Reload your shell:

```bash
source ~/.zshrc   # macOS (zsh)
# or
source ~/.bashrc  # Linux (bash)
```

Verify nvm loaded (do not use `which nvm` — nvm is a shell function):

```bash
command -v nvm
# => nvm
```

### Install Node LTS

```bash
# Install Node 22 (current LTS as of 2025)
nvm install 22

# Use it in the current shell session
nvm use 22

# Set as the default for new terminal sessions
nvm alias default 22
```

### Verify Node

```bash
node -v
# => v22.x.x

npm -v
# => 10.x.x
```

## Install Yarn

Rails integrates with Yarn via `jsbundling-rails` and `cssbundling-rails`. Use Corepack (included with Node 16+) to manage Yarn — do not install Yarn with npm.

```bash
# Enable Corepack (ships with Node, but may need activation)
corepack enable

# Install and activate the latest stable Yarn
corepack prepare yarn@stable --activate

# Verify
yarn -v
# => 4.x.x  (Yarn Berry/Modern)
```

If your project uses Yarn Classic (1.x) — check `package.json` for `"packageManager"` or ask your team:

```bash
corepack prepare yarn@1 --activate
yarn -v
# => 1.22.x
```

## Pin Node Version Per Project

Create a `.nvmrc` file in your project root to ensure everyone uses the same Node version:

```bash
echo "22" > .nvmrc
```

Then anyone cloning the project can run:

```bash
nvm use   # reads .nvmrc automatically
```

## Verify the Full Setup

```bash
node -v    # => v22.x.x
npm -v     # => 10.x.x
yarn -v    # => 4.x.x (or 1.22.x for classic)
```

## Common Issues

### `nvm: command not found` after install

nvm did not initialize. Check your shell config file for the nvm lines:

```bash
grep -n nvm ~/.zshrc   # or ~/.bashrc
```

If the lines are missing, add them manually:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

Then `source ~/.zshrc` and try again.

### macOS: `nvm install` fails with SSL errors

Ensure Homebrew's OpenSSL is installed:

```bash
brew install openssl
```

### `corepack enable` requires sudo on some Linux distros

If Node was installed globally (not via nvm), corepack may be in a protected directory. With nvm, your Node install lives in `~/.nvm` and never requires sudo.

---

## Next Step

Set up your editor: see `editor-setup.md`.
