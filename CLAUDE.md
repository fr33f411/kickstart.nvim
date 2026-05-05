# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal Neovim config, forked from `nvim-lua/kickstart.nvim`. Loads from `~/.config/nvim`. Targets latest stable/nightly Neovim. There are no unit tests — validate changes by launching Neovim and using `:checkhealth` and `:Lazy`.

`AGENTS.md` already exists with a condensed version of the same conventions. Keep both files in sync if you change one.

## Commands

| Task | Command |
|---|---|
| Format Lua | `stylua .` |
| Lint (CI check) | `stylua --check .` |
| Sync plugins | `:Lazy sync` (or `:Lazy update`) |
| Inspect plugin state | `:Lazy` |
| Manage LSP/tools | `:Mason` |
| Verify environment | `:checkhealth` |
| Format current buffer | `<leader>f` (conform.nvim) |

`stylua` config: 2-space indent, 160-col width, `AutoPreferSingle` quotes, `call_parentheses = "None"` (no parens on single-arg calls), Unix line endings. CI runs `stylua --check .` on PRs (`.github/workflows/stylua.yml`) — keep formatting clean before committing.

## Architecture

**Single-file kickstart layout.** Almost everything lives in `init.lua` (~1180 lines). Plugin manager is `lazy.nvim`, auto-bootstrapped at the top of `init.lua` into `stdpath('data')/lazy/lazy.nvim`. The single `require('lazy').setup({ ... })` call holds every plugin spec inline. `lazy-lock.json` is tracked.

Reading order inside `init.lua`:

1. Leader keys + `vim.g.have_nerd_font` (line ~90)
2. `vim.o` / `vim.opt` options (line ~96)
3. Basic keymaps + `TextYankPost` autocmd (line ~169)
4. lazy.nvim bootstrap (line ~222)
5. Plugin specs inside `require('lazy').setup({ ... })` (line ~248 to ~1175)

**Plugin spec conventions.** Use the `{ 'owner/repo', opts = {...} }` table form so lazy calls `require(MODULE).setup(opts)` automatically. Use `config = function() ... end` only when you need imperative setup (e.g. lspconfig handler wiring, custom keymaps tied to plugin loading). Lazy-load with `event`, `cmd`, `ft`, or `keys` where it makes sense — but historically some plugins were intentionally not lazy-loaded (e.g. neo-tree, see commit `fb73617`) to keep startup features like netrw hijacking working.

**Where new code goes.**

- Tweaks to existing plugins or core options → edit `init.lua` directly.
- New user plugins → add files under `lua/custom/plugins/*.lua` and uncomment `{ import = 'custom.plugins' }` in `init.lua` (currently commented out around line 1149). `lua/custom/plugins/init.lua` returns `{}`; the `custom/plugins` directory is reserved for the user and is excluded from upstream merge conflicts.
- Optional kickstart-provided plugins (`debug`, `indent_line`, `lint`, `autopairs`, `neo-tree`, `gitsigns` extras) live in `lua/kickstart/plugins/*.lua` and are enabled by uncommenting the matching `require 'kickstart.plugins.X'` line near the bottom of `init.lua` (~lines 1138–1143).
- Health check logic lives in `lua/kickstart/health.lua` and is invoked by `:checkhealth kickstart`.

**Subsystems already wired up in `init.lua`.**

- LSP: `neovim/nvim-lspconfig` + `mason.nvim` + `mason-lspconfig.nvim` + `mason-tool-installer.nvim`. Add new servers to the `servers` table inside the `nvim-lspconfig` `config` function — `mason-tool-installer` reads `vim.tbl_keys(servers)` plus the extras list (currently `stylua`) to decide what to install. Per-buffer LSP keymaps are set on `LspAttach` (`grn`, `gra`, `grr`, `gri`, `grd`, `grD`, `gO`, `gW`, `grt`, `<leader>th`).
- Completion: `saghen/blink.cmp` (capabilities are merged into LSP capabilities via `require('blink.cmp').get_lsp_capabilities()`).
- Formatting: `stevearc/conform.nvim` runs `format_on_save` (timeout 500 ms, `lsp_format = 'fallback'`); `c` and `cpp` are explicitly disabled.
- Pickers: `nvim-telescope/telescope.nvim` with `fzf-native` and `ui-select` extensions. Many `<leader>s*` and LSP mappings funnel through Telescope.
- Treesitter: `nvim-treesitter` on `master` branch, `:TSUpdate` build, `auto_install = true`, `additional_vim_regex_highlighting = { 'ruby' }`.
- Misc: `which-key`, `gitsigns` (with current-line blame enabled), `trouble.nvim`, `undotree` (`<leader>u`), `mini.nvim` (statusline + textobjects + surround), `lazydev.nvim` for Lua/Neovim API completion, `catppuccin/nvim` colorscheme, `todo-comments.nvim`.

**Keymap conventions.**

- Leader is `<space>`; localleader is `<space>`.
- Use `vim.keymap.set` with a `desc` field — which-key surfaces these (e.g. `[S]earch`, `[F]ormat`).
- Window nav: `<C-h/j/k/l>`. Terminal escape: `<Esc><Esc>`.
- LSP keymaps are scoped to the buffer in the `LspAttach` callback.

## Working notes

- `:help` is the source of truth — many comment blocks point at `:help <topic>` and the doc tags ship in `doc/kickstart.txt` / `doc/tags`.
- The kickstart upstream goal is "single-file, fully documented." Keep heavy explanatory comments when editing core sections; brief comments are fine for user-only additions.
- Prefer `vim.o` / `vim.opt` (recent commit `c92ea7c` migrated away from `vim.opt` for scalars) and `vim.hl.on_yank` over the deprecated `vim.highlight.on_yank` (commit `6ba2408`).
- Errors and user notices: use `vim.notify(msg, vim.log.levels.X)`.
