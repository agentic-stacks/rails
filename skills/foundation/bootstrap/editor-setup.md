# Set Up Your Editor

A properly configured editor gives you autocompletion, inline errors, go-to-definition, and formatting for Ruby and Rails. This file covers VS Code, RubyMine, and Vim/Neovim, plus a universal `.editorconfig` that applies to any editor.

## Standard `.editorconfig`

Add this to the root of every Rails project. All major editors respect it automatically (VS Code needs the EditorConfig extension).

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab
```

Rails uses 2-space indentation everywhere by default. This config enforces it across Ruby, ERB, JavaScript, YAML, and JSON files.

---

## VS Code

### Required Extensions

Install these from the Extensions panel (`Cmd+Shift+X` / `Ctrl+Shift+X`) or via the CLI:

```bash
# Ruby language server — maintained by Shopify
code --install-extension Shopify.ruby-lsp

# EditorConfig support
code --install-extension EditorConfig.EditorConfig
```

### Recommended Extensions

```bash
# Rails-specific snippets and navigation
code --install-extension bung87.rails

# ERB syntax highlighting and formatting
code --install-extension aliariff.vscode-erb-beautify

# Stimulus LSP (autocomplete for Stimulus controllers)
code --install-extension marcoroth.stimulus-lsp

# Prettier for formatting JS/CSS (if using jsbundling/cssbundling)
code --install-extension esbenp.prettier-vscode
```

### VS Code Settings

Add to your project's `.vscode/settings.json` (create the file if it does not exist):

```json
{
  "[ruby]": {
    "editor.defaultFormatter": "Shopify.ruby-lsp",
    "editor.formatOnSave": true,
    "editor.tabSize": 2
  },
  "[erb]": {
    "editor.tabSize": 2
  },
  "rubyLsp.formatter": "rubocop",
  "rubyLsp.enabledFeatures": {
    "diagnostics": true,
    "formatting": true,
    "codeActions": true,
    "hover": true,
    "inlayHint": true,
    "completion": true,
    "definition": true
  },
  "editor.rulers": [120],
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true
}
```

### Enable ruby-lsp in Your Project

ruby-lsp works best when added to your project's `Gemfile` in the development group:

```ruby
# Gemfile
group :development do
  gem "ruby-lsp"
  gem "rubocop"           # optional, enables format-on-save
  gem "rubocop-rails"     # optional, Rails-specific cops
end
```

Then run:

```bash
bundle install
```

VS Code will automatically use the bundled `ruby-lsp` instead of a global one, ensuring the version matches your project.

---

## RubyMine

RubyMine (JetBrains) has built-in Ruby and Rails support — no extensions required.

### Initial Configuration

1. Open your Rails project: `File → Open` → select the project directory.
2. RubyMine detects the `Gemfile` and prompts to run `bundle install`. Accept.
3. Set the SDK: `RubyMine → Preferences → Languages & Frameworks → Ruby SDK and Gems` → select the rbenv/asdf Ruby you installed.
4. Enable Rails support: `Preferences → Languages & Frameworks → Rails` → check "Enable Rails support".

### Useful RubyMine Settings

- **Code style**: `Preferences → Editor → Code Style → Ruby` — set indent to 2 spaces.
- **Line endings**: `Preferences → Editor → Code Style` → set "Line separator" to `Unix and macOS (\n)`.
- **Inspections**: `Preferences → Editor → Inspections → Ruby` — enable RuboCop integration.

---

## Vim / Neovim

### vim-rails

vim-rails adds Rails-aware navigation: jump between model, controller, view, test, and migration with `:Emodel`, `:Econtroller`, `:Eview`, and related commands.

```vim
" Using vim-plug (add to your init.vim or .vimrc)
Plug 'tpope/vim-rails'
Plug 'tpope/vim-bundler'
```

Then run `:PlugInstall`.

### LSP with ruby-lsp (Neovim)

Configure Neovim's built-in LSP client to use ruby-lsp. Using `nvim-lspconfig`:

```lua
-- In your Neovim config (init.lua or lua/lsp.lua)
require('lspconfig').ruby_lsp.setup({
  init_options = {
    formatter = 'rubocop',
    linters = { 'rubocop' },
  },
  on_attach = function(client, bufnr)
    -- Enable format on save
    vim.api.nvim_create_autocmd("BufWritePre", {
      buffer = bufnr,
      callback = function()
        vim.lsp.buf.format({ async = false })
      end,
    })
  end,
})
```

Install via your plugin manager (lazy.nvim example):

```lua
{
  "neovim/nvim-lspconfig",
  config = function()
    -- paste config above here
  end,
}
```

### ruby-lsp must be in PATH or Gemfile

Neovim's LSP starts ruby-lsp as a subprocess. Either:

- Add `gem "ruby-lsp"` to your `Gemfile` and use a wrapper like `bundle exec ruby-lsp`, or
- Install globally: `gem install ruby-lsp`

The Gemfile approach is recommended for version consistency.

---

## Verify Your Editor Setup

Open a Rails project and check:

- [ ] Ruby files show syntax highlighting
- [ ] Hover over a method shows its signature or documentation
- [ ] Go-to-definition works (`F12` in VS Code, `gd` in Neovim, `Cmd+B` in RubyMine)
- [ ] Saving a Ruby file triggers formatting (if RuboCop is configured)
- [ ] `.editorconfig` rules are respected (2-space indent, LF line endings)

---

## Next Step

Your environment is fully set up. Create your first Rails application: see `skills/operations/setup/new-app.md`.
