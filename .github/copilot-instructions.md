# GitHub Copilot Workspace Instructions

This file provides GitHub Copilot-specific guidance for the `output-panel.nvim` repository.

## Core Guidance

For architectural decisions, coding standards, and general contribution guidelines, see [AGENTS.md](../AGENTS.md) in the repository root. This file contains only Copilot-specific instructions and workspace context.

## Workspace Context

- **Language**: Lua (Neovim plugin)
- **Primary file**: `lua/output-panel/init.lua`
- **Dependencies**: VimTeX (optional adapter), Overseer.nvim (optional adapter), snacks.nvim (optional), Neovim 0.8+
- **Formatting**: StyLua (run `stylua lua/` before committing)

## Copilot-Specific Best Practices

### Code Suggestions

When suggesting code completions:

1. **Match existing patterns**: The codebase uses specific patterns for state management, window creation, event handling, and command execution. Follow these patterns rather than introducing new approaches.

2. **Preserve comments**: When modifying functions, maintain or enhance existing comments. Comments explain *why*, not just *what*.

3. **Window configuration**: Window dimension calculations are complex. When suggesting changes to `window_dimensions()` or related functions, consider all edge cases (small terminals, large monitors, different anchors).

4. **State token patterns**: The codebase uses token-based cancellation (e.g., `hide_token`, `render_retry_token`). When suggesting async operations, follow this pattern.

5. **Generic API first**: The plugin's core is a generic command runner (`run()`). Adapters (VimTeX, Overseer) and helpers (Make) are optional integrations. When suggesting new features, consider whether they belong in the core runner or as an adapter.

### Testing Suggestions

Suggest manual testing approaches when relevant:

- Test with and without snacks.nvim installed
- Test with and without VimTeX installed (VimTeX adapter is optional)
- Test with and without Overseer installed (Overseer adapter is optional)
- Test auto-open enabled and disabled
- Test with various window sizes
- Test rapid compilation triggers (save spam)
- Test with missing or inaccessible log files
- Test generic `run()` command execution with various shell commands
- Test Make helper with different `makeprg` configurations
- Test profile system with custom profiles and overrides

### Documentation

When suggesting documentation changes:

- Ensure README examples match actual code defaults
- Add troubleshooting entries for edge cases you discover
- Update the Architecture and Configuration sections if behavior changes affect core APIs
- Keep the "How does this compare" section accurate when adding features similar to other plugins

## Quick Reference

### Key Public Functions

- `M.setup(opts)`: Main entry point, merges user config with defaults
- `M.run(opts)`: Execute arbitrary shell commands with live output streaming
- `M.stream(opts)`: Lower-level API for custom adapters to stream output
- `M.make(args)`: Run Neovim's makeprg through the panel (`:Make` command)
- `M.show()`: Open the panel for the current target
- `M.hide()`: Close the panel
- `M.toggle()`: Toggle panel visibility
- `M.toggle_follow()`: Toggle follow/tail mode
- `M.toggle_focus()`: Switch between mini and focus modes
- `M.adapter_enabled(name)`: Check if a profile/adapter is enabled

### Key Internal Functions

- `render_window(opts)`: Creates or updates the floating window
- `update_buffer_from_file(force)`: Polls log file and updates buffer
- `start_polling()`: Initiates live update loop
- `on_compile_*()`: Event handlers for VimTeX compilation events
- `window_dimensions()`: Calculates window size and position
- `notifier()`: Returns the active notification backend
- `current_config()`: Returns the merged configuration for current context

### State Variables

- `state.win`: Window handle (or nil if closed)
- `state.buf`: Buffer handle for log content
- `state.buffer_targets`: Map of buffer numbers to their target output paths
- `state.status`: "idle" | "running" | "success" | "failure"
- `state.focused`: Boolean for mini vs. focus mode
- `state.follow`: Boolean for tail/follow mode
- `state.timer`: uv timer for polling loop
- `state.target`: Current output file path
- `state.job`: Active command job handle
- `state.active_config`: Merged configuration for current context
- `state.failure_notifications`: Map of scoped failure notifications
- `state.hide_token`: Cancellation token for auto-hide
- `state.render_retry_token`: Cancellation token for render retries

### Configuration Sections

- `config.mini`: Compact overlay dimensions (unfocused mode)
- `config.focus`: Expanded overlay dimensions (interactive mode)
- `config.auto_open`: Auto-show panel when commands/builds start
- `config.auto_hide`: Auto-hide panel after successful builds
- `config.notifications`: Notification behavior (titles, persistence for errors)
- `config.follow`: Auto-scroll to bottom of output (tail mode)
- `config.poll`: How often to refresh output while panel is visible
- `config.max_lines`: Maximum buffer lines before trimming
- `config.open_on_error`: Force panel open when commands fail
- `config.notifier`: Custom notification backend
- `config.profiles`: Named configuration presets (vimtex, overseer, make, custom)
- `config.scrolloff_margin`: Extra cursor padding while mini mode is active
- `config.border_highlight`: Highlight group for window border

## Common Pitfalls

1. **Don't break state isolation**: The module maintains a single global state. Avoid introducing module-level variables that bypass state management.

2. **Don't assume synchronous operations**: File polling, timers, autocmds, and job execution are all asynchronous. Use tokens for cancellation and avoid race conditions.

3. **Don't hardcode dimensions**: All window dimensions come from config. Don't suggest magic numbers; use config values or calculated dimensions.

4. **Don't forget error handling**: pcall is used extensively for Neovim API calls. Maintain this pattern for robustness.

5. **Don't mix adapter concerns with core logic**: The generic command runner should work standalone. Adapter-specific logic belongs in adapter modules or event handlers, not in core functions.

6. **Don't assume dependencies**: VimTeX, Overseer, and snacks.nvim are all optional. Code must gracefully handle their absence.

## Integration Points

This plugin integrates with:

- **VimTeX** (optional adapter): Reads `vim.b.vimtex.compiler.output` for log file path, listens to VimTeX autocmd events (`VimtexEventCompiling`, `VimtexEventCompileSuccess`, `VimtexEventCompileFailed`)
- **Overseer.nvim** (optional adapter): Provides `output_panel` component that tasks can opt into for streaming output
- **snacks.nvim** (optional): Optional dependency for enhanced notifications (falls back to `vim.notify`)
- **Neovim APIs**: Uses floating windows, buffers, timers, autocmds, and job control (`vim.fn.jobstart`/`vim.system`)

When suggesting changes, be mindful of these integration boundaries and test compatibility. Adapters should be non-invasive and respect the `enabled` flag in their respective profiles.

## Commit Message Format (MANDATORY)

All commit messages **must** follow this exact format:
```
(scope) Brief description of the change
```

**Rules:**
- The scope **must** be a single word enclosed in parentheses
- The scope should be lowercase
- Use a compound word (e.g., `auto-hide`, `config-merge`) only if absolutely necessary
- There must be exactly one space between the closing parenthesis and the description
- The description should be concise and written in imperative mood

**Common scopes:**
- `docs` - Documentation changes
- `config` - Configuration-related changes
- `window` - Window rendering or positioning
- `notify` - Notification system changes
- `state` - State management changes
- `poll` - Polling mechanism changes
- `event` - Event handler changes
- `fix` - Bug fixes
- `refactor` - Code refactoring
- `meta` - Repository meta files or tooling

**Examples:**
- `(docs) Fix default value inconsistencies in README`
- `(config) Add auto_hide option for successful builds`
- `(window) Adjust mini mode dimensions for small terminals`
- `(notify) Implement persist_failure configuration`
