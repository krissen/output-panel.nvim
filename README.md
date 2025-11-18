# output-panel.nvim

A configurable floating output window for Neovim jobs. The plugin began life as a
VimTeX helper but now exposes a generic runner so any long-running shell
command—`Rscript`, `pandoc`, custom build scripts, etc.—can stream their stdout
and stderr into the same Snacks-friendly panel while surfacing success/failure
notifications. The original VimTeX integration remains available as a
preconfigured profile and example workflow.

## Requirements

- Neovim **0.10+** (`vim.system` streaming is required for the generic runner)
- [snacks.nvim](https://github.com/folke/snacks.nvim) *(optional)* — richer
  notifications and theming
- [VimTeX](https://github.com/lervag/vimtex) *(optional)* — only needed if you
  want automatic LaTeX compiler integration

## Highlights

- **General runner** – `require("output-panel").run()` executes any shell
  command asynchronously, streams live output, colours the border based on exit
  status, and fires notifications.
- **Profiles & overrides** – Define reusable configuration profiles and apply
  them per command so each workflow can tweak window geometry, notification
  titles, auto-hide timing, etc.
- **Snacks-aware UI** – Falls back to `vim.notify` but automatically upgrades to
  `snacks.notify` when available. You can also provide a custom notifier.
- **VimTeX aware** – Ships the original auto-open, auto-hide, and command suite
  for `latexmk` logs, now powered by the same configurable panel.

## Installation

### lazy.nvim

```lua
{
  "krissen/output-panel.nvim",
  dependencies = {
    -- Optional, but recommended for LaTeX workflows
    "lervag/vimtex",
    -- Optional notifications/popup helpers
    "folke/snacks.nvim",
  },
  config = function()
    require("output-panel").setup({
      -- global options go here; see below for the full list
    })
  end,
}
```

> Existing configs that still `require("snacks-vimtex-output")` continue to
> work via a compatibility shim, but new setups should switch to
> `require("output-panel")`.

## Usage

### Running arbitrary commands

Call `run()` with the command you want to execute. Output streams into the
panel, and notifications summarise the exit status.

```lua
local panel = require("output-panel")

-- Knit the current RMarkdown file using Rscript
vim.keymap.set("n", "<leader>rk", function()
  panel.run({
    cmd = { "Rscript", "bifrost__knit.R", "-r", vim.fn.expand("%") },
    profile = "knit", -- optional profile defined in setup()
    success = "Document knitted",
    error = "Knitting failed",
    start = "Knitting…",
  })
end, { desc = "Knit document" })
```

Key behaviours:

- `cmd` accepts a string, list, or function returning either. Strings run inside
  `sh -c` (or `cmd.exe /c` on Windows).
- Notifications inherit their title/timeouts from the active profile. Provide
  `success`/`error` strings for custom labels.
- Set `open = false` to keep the panel hidden until you manually call
  `require("output-panel").show()`.
- Each run gets its own temporary log file, so you can revisit the output even
  after the command completes.

### VimTeX integration

Install VimTeX, call `setup()`, and the panel automatically follows
`VimtexEventCompiling`, `VimtexEventCompileSuccess`, etc. The legacy commands are
still available:

| Command | Description |
| --- | --- |
| `:VimtexOutputShow` | Open the panel for the active VimTeX buffer. |
| `:VimtexOutputHide` | Close the panel. |
| `:VimtexOutputToggle` | Toggle visibility. |
| `:VimtexOutputToggleFocus` | Switch between mini/focus layouts. |

The overlay attaches to VimTeX's compiler log (usually the `latexmk` stdout
buffer) and retains all prior behaviours—auto-open on compilation, optional
auto-hide on success, green/red borders, and persistent error notifications.

### API helpers

```lua
local panel = require("output-panel")
panel.show()          -- open panel for the current target
panel.hide()          -- hide panel
panel.toggle()        -- toggle visibility
panel.toggle_focus()  -- swap between mini/focus layouts
panel.run({...})      -- run arbitrary commands (see above)
```

## Configuration

`setup()` merges your overrides with the defaults below. Profiles are deep
merged, so you can inherit global values and tweak only what each workflow
needs.

```lua
require("output-panel").setup({
  mini = {
    width_scale = 0.90,
    width_min = 48,
    width_max = 120,
    height_ratio = 0.10,
    height_min = 5,
    height_max = 14,
    row_anchor = "bottom",
    row_offset = 5,
    horizontal_align = 0.55,
    col_offset = 0,
  },
  focus = {
    width_scale = 0.95,
    width_min = 70,
    width_max = 220,
    height_ratio = 0.6,
    height_min = 14,
    height_max = 0,
    row_anchor = "center",
    row_offset = -1,
    horizontal_align = 0.5,
    col_offset = 0,
  },
  scrolloff_margin = 5,
  border_highlight = "FloatBorder",
  auto_open = {
    enabled = false,
    retries = 6,
    delay = 150,
  },
  auto_hide = {
    enabled = true,
    delay = 3000,
  },
  notifications = {
    enabled = true,
    title = "VimTeX",
    persist_failure = true,
  },
  notifier = nil,
  profiles = {
    knit = {
      notifications = { title = "Knit" },
      auto_hide = { enabled = false },
    },
  },
})
```

### Profiles

Profiles are arbitrary tables merged into the active configuration whenever you
call `run({ profile = "name" })`. Use them to change the notification title,
auto-hide behaviour, or window layout for a specific workflow without touching
other commands.

You can also bypass profiles and pass `config = { ... }` directly to `run()` for
one-off overrides.

### Notification backends

Notifications follow a fallback chain:

1. Custom notifier provided via `setup({ notifier = { info=…, warn=…, error=… } })`
2. `snacks.notify` (if Snacks is installed and exposes it)
3. `vim.notify`

This applies everywhere—VimTeX events and manual command runs.

### Tips

- Increase `mini.height_ratio` or `mini.height_max` if your commands are chatty.
- Bump `scrolloff_margin` if the mini panel covers your cursor.
- Set `auto_open.enabled = true` to auto-pop the panel whenever VimTeX begins
  compiling.
- Disable `auto_hide` globally or per profile if you want successful runs to
  stay visible.
- Assign `notifier` to integrate with `noice.nvim`, `rcarriga/nvim-notify`, or
  any custom logger.

## Troubleshooting

- **Nothing shows up when running a command** – Ensure you're on Neovim 0.10+
  and that the command exists in your `$PATH`. The panel writes every chunk to a
  temp file; open it via `:edit {path}` to inspect raw output.
- **VimTeX overlay never opens** – Set `auto_open.enabled = true` or run one of
  the `:VimtexOutput*` commands manually. Verify `vim.b.vimtex.compiler.output`
  is populated in your TeX buffer.
- **Notifications missing** – Confirm `notifications.enabled = true` and that
  your custom notifier isn't erroring. The plugin logs failures inside the panel.
- **Want different layouts per workflow** – Define multiple profiles and pass
  `profile = "name"` when calling `run()`.

## Why keep the VimTeX docs?

The project started as *snacks-vimtex-output*, and the LaTeX workflow remains a
first-class citizen. VimTeX already streams compiler logs to splits, but the
panel provides:

| Feature | Panel | VimTeX stock output |
| --- | --- | --- |
| Display | Floating mini/focus overlay | Regular splits |
| Auto-hide | Yes (configurable) | No |
| Visual status | Border colours + notifications | Messages only |
| Buffer type | Scratch, hidden by default | Normal buffer |

Use whichever suits your workflow—the panel simply adds a Snacks-friendly skin
on top of VimTeX's reliable compilation backend while now powering any other
command you want to monitor.

## Development

- Format with `stylua lua/`.
- Run different commands/profiles manually to test notification paths.
- Verify behaviour with and without Snacks and VimTeX loaded.
