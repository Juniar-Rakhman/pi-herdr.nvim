# Repository Guidelines

## Project Overview

`pi-herdr.nvim` is a minimal Neovim plugin (Lua) that bridges Neovim to an already-running [Pi](https://github.com/earendil-works/pi-coding-agent) coding agent managed by [Herdr](https://github.com/ctbaum/herdr). It never starts a new Pi process; it matches agents by exact canonicalized CWD via `herdr agent list`, then sends compact `@file:line`/range context through `herdr agent send-keys` or `herdr agent prompt`. Supports `snacks.input` with a custom float fallback.

## Architecture & Data Flow

Single-module plugin. All logic lives in `lua/pi-herdr/init.lua`, returned as table `M`.

Data flow:
1. `setup(opts)` merges config, registers `:Pi*` commands and keymaps.
2. Entry points (`ask_normal`, `ask_visual`, `ask_buffer`, `select`) call `M.ask(prefix, opts)`.
3. `ask` snapshots editor context via `capture_context` *before* opening the prompt float (so line numbers survive the float), then `open_prompt`.
4. On submit, `substitute_placeholders` expands `@this`/`@buffer`/`@diagnostics` using the snapshot.
5. `find_pi_agent` spawns `herdr agent list`, filters `agent == "pi"` with canonicalized CWD match; prompts on multiple matches.
6. `deliver` spawns herdr (`agent prompt` if `submit_after_send`, else `agent send-keys` with per-char key args), optionally `agent focus`.

Async boundary: all herdr calls use `vim.uv.spawn` with JSON stdout; callbacks are wrapped in `vim.schedule` before touching Vimscript/UI (`find_pi_agent`). `deliver` uses raw spawn without JSON parsing.

## Key Directories

- `lua/pi-herdr/` — sole source directory; `init.lua` is the entire plugin.
- Root: `README.md` (user docs), `ROADMAP.md` (out-of-scope features).

No tests, build scripts, or CI present.

## Development Commands

No build step. Plugin is loaded by a plugin manager (lazy.nvim) or `:source`. Manual smoke testing in Neovim:

```sh
# Syntax check Lua (if luacheck/stylua installed)
luacheck lua/ 2>/dev/null || true
stylua --check lua/ 2>/dev/null || true

# Run headless smoke (example)
nvim --headless -c "lua require('pi-herdr').setup({})" -c "qa"
```

Lint/format tooling not configured in repo; follow existing style.

## Code Conventions & Common Patterns

- Lua 5.1/LuaJIT (Neovim). Use `vim.*` APIs, `vim.uv` (libuv) for process spawn.
- Module pattern: `local M = {}`, functions on `M`, `return M`.
- Private helpers are `local function`; public API on `M` (e.g. `M.build_context_normal`).
- Config lives in `M.config` table with defaults; merged via `vim.tbl_deep_extend("force", ...)`.
- Async callbacks: wrap Vimscript/UI calls in `vim.schedule`. Keep process spawn args as arrays.
- Path comparison: always canonicalize via `M.canonical_path` (absolute, resolved, trailing slash stripped) before comparing Neovim vs herdr CWDs.
- Context capture must happen before UI opens; reuse `capture_context`/`substitute_placeholders` rather than re-reading editor state later.
- Placeholders are literal `@this`, `@buffer`, `@diagnostics` (gsub). Preserve trailing newline in context builders.
- Error handling: `pcall` for optional requires (`snacks`); `vim.notify` with `vim.log.levels`.
- Naming: snake_case functions/locals; config keys snake_case. Commands `Pi*` PascalCase.

## Important Files

- `lua/pi-herdr/init.lua` — entire plugin (config, herdr spawn, agent matching, context builders, prompt UI, commands/keymaps).
- `README.md` — user-facing docs/config/commands.
- `ROADMAP.md` — explicitly out-of-scope features; do not implement unless asked.

## Runtime/Tooling Preferences

- Runtime: Neovim 0.10+ (uses `vim.uv`, `vim.json`, `vim.tbl_*`).
- External dependency: `herdr` binary on PATH (or `herdr_binary` config).
- Package manager: none (plugin manager on user side). No `package.json`/rockspec.

## Testing & QA

No automated test suite. Verify changes by:
- Headless load: `nvim --headless -c "lua require('pi-herdr').setup({})" -c "qa"`.
- Manual: open Neovim in a dir with a running Pi agent via Herdr, trigger `:PiAsk`/visual `:PiAskVisual`, confirm context format and delivery.
- When adding exported helpers, keep them on `M` and prefer existing patterns (`build_context_*`, `capture_context`).
- Do not add tests unless requested; use throwaway smoke checks.
