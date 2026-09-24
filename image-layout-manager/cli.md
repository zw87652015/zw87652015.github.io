# `imagelayout-cli`

Headless driver and MCP adapter for ImageLayoutManager. Render, pack,
inspect, **edit**, and let AI hosts control the running GUI through
`imagelayout-cli.exe mcp`.

## Verbs

| Verb      | Purpose                                                         |
| --------- | --------------------------------------------------------------- |
| `render`  | `.figpack` / `.figlayout` / `.json` → `pdf` / `tiff` / `jpg` / `png` (writes the output path to stdout) |
| `pack`    | `.figlayout` / `.json` → `.figpack` (bundle layout + referenced assets)   |
| `unpack`  | `.figpack` → folder containing assets + sidecar `.figlayout`    |
| `inspect` | Print page, DPI, layout mode, cells, labels, group labels, size groups, export region (text or `--json`) |
| `edit`    | Apply agent tool operations headlessly — the same 39 tools as the MCP adapter |
| `mcp`     | Stdio MCP adapter for AI hosts; proxies 39 layout/styling/export tools to the running GUI |

## Examples

```powershell
# Pixel-perfect PDF render at the project's saved DPI
imagelayout-cli.exe render figure_4.figpack -f pdf -o figure_4.pdf

# Override DPI for a fast preview PNG
imagelayout-cli.exe render figure_4.figlayout -f png --dpi 150

# Print-ready CMYK TIFF using a specific ICC profile
imagelayout-cli.exe render figure_4.figpack -f tiff --cmyk `
    --icc-profile "C:\ICC\USWebCoatedSWOP.icc" --icc-intent 1 -o fig.tiff

# Bundle a .figlayout + every referenced image into a .figpack
imagelayout-cli.exe pack figure_4.figlayout -o figure_4.figpack

# Unpack a .figpack so you can edit the JSON / images by hand
imagelayout-cli.exe unpack figure_4.figpack -o ./extracted/

# Quick summary
imagelayout-cli.exe inspect figure_4.figpack
imagelayout-cli.exe inspect figure_4.figpack --json

# Headless edit — auto-label cells in place
imagelayout-cli.exe edit figure_4.figlayout --in-place --call auto_label_cells '{"scheme": "(a)"}'

# Apply a script of steps, pack the result, get a JSON report
imagelayout-cli.exe edit figure_4.figlayout -o out.figpack --script ops.json --json

# Start from a blank project and add a row of three cells
imagelayout-cli.exe edit --new -o blank.figlayout --call row_add '{"position": 2, "column_count": 3}'

# Pipe a script on stdin, stream the edited .figlayout JSON to stdout
cat ops.json | imagelayout-cli.exe edit figure_4.figlayout -o - --script -

# MCP adapter used by Claude, Cursor, Windsurf, Cline, etc.
imagelayout-cli.exe mcp
```

## Headless editing (`edit`)

`edit` runs the same 39 agent tools an MCP host drives — against a
project file, with no GUI running at all. Steps are applied in order:
`--script` first, then each `--call`. Nothing is written unless every
step succeeds; use `--keep-going` to continue past a failing step and
still write the result (the exit code stays 1).

| Option              | Purpose                                                          |
| ------------------- | ---------------------------------------------------------------- |
| `INPUT`             | `.figlayout` or `.json` — omit when using `--new`                 |
| `--new`             | Start from a blank project instead of `INPUT`                    |
| `--call TOOL`       | Run `TOOL`, optionally followed by a JSON object of params. Repeatable; applied in order. |
| `--script FILE`     | JSON array of `{"tool": ..., "params": {...}}` steps (`-` reads stdin) |
| `-o`, `--output`    | Write the result: `.figlayout`, `.json` or `.figpack` (`-` streams `.figlayout` JSON to stdout) |
| `-i`, `--in-place`  | Overwrite `INPUT` (written atomically)                           |
| `--dry-run`         | Apply every step in memory but write nothing                     |
| `--keep-going`     | Continue past a failing step and still write the result          |
| `--json`            | Emit one JSON object containing every step envelope              |
| `--list-tools`      | Print the available tool names and exit                          |

Notes:

- `.figpack` is **not** accepted as `edit` input — run
  `imagelayout-cli unpack in.figpack -o dir` first, edit
  `dir/<name>.figlayout`, then `pack` it back. Re-packing straight from
  a `.figpack`'s temp working dir would silently drop assets whose
  original source is missing on this machine.
- `--script` accepts a bare JSON array or `{"steps": [...]}`; each step
  may use `tool`/`name` and `params`/`args` as aliases, so hand-written
  scripts and tool-call logs both work. A UTF-8 BOM is tolerated
  (PowerShell's `Out-File` writes one by default).
- Like `render`, stdout carries only the output path (or the `--json`
  report) so the verb stays pipeable; per-step progress goes to stderr.

### The 39 agent tools

| Category              | Tools |
| --------------------- | ----- |
| Project lifecycle     | `project_describe` `project_new` `project_open` `project_save` `project_export` |
| Layout topology       | `row_add` `row_remove` `row_set` `cell_add` `cell_remove` `cell_swap` `cell_split` `cell_set_split_ratios` `layout_set_mode` |
| Image content         | `image_import` `cell_set_geometry` `cell_set_properties` `cell_set_scale_bar` `cell_set_z_index` |
| PiP insets            | `pip_add` `pip_remove` `pip_set_properties` |
| Text & labels         | `text_add` `text_remove` `text_set_style` `labels_set_style` `project_set_label_style` `auto_label_cells` |
| Group labels          | `group_label_add` `group_label_remove` `group_label_set` |
| Size groups           | `size_group_create` `size_group_delete` `size_group_set` `size_group_assign` |
| Export region         | `export_region_set` `export_region_clear` |
| Algorithmic + vision  | `auto_layout` `view_screenshot` |

Run `imagelayout-cli edit --list-tools` to print the current list at any
time.

## MCP automation

The `mcp` verb is a stdio adapter for MCP-compatible AI hosts. It connects
to the running ImageLayoutManager GUI over localhost WebSocket (reading
port + token from a local discovery file), so the AI can drive the same
39 tools `edit` exposes: create layouts, import images, style
labels/text, crop/rotate/pad panels, add scale bars, add PiP insets,
manage size groups, set export regions, request screenshots, and
save/export projects.

In the GUI, enable **Tools → Enable MCP Server** (or tick
**Auto-start MCP Server on launch** in Preferences, or launch with
`ImageLayoutManager.exe --agent-server`). Then configure your AI
host to run:

```json
{
  "mcpServers": {
    "imagelayout": {
      "command": "C:/Program Files/ImageLayoutManager/imagelayout-cli.exe",
      "args": ["mcp"]
    }
  }
}
```

After upgrading ImageLayoutManager, restart both the app and your AI host.
If the installer reports a locked `.pyd` file, fully quit the AI host or
stop any remaining `imagelayout-cli.exe` process, then run the installer
again.

## Exit codes

| Code | Meaning                                            |
| ---- | -------------------------------------------------- |
| 0    | Success                                            |
| 1    | User-facing error (bad path, unknown format, a failed `edit` step, etc.) |
| 2    | Argparse usage error                               |
| 3    | Bundle integrity / security failure                |
| 4    | Unexpected internal error                          |
| 130  | Interrupted (Ctrl+C)                               |

Set `IMAGELAYOUT_CLI_DEBUG=1` to print full tracebacks on exit-4 errors.

## Output parity

`render` reuses the same `PdfExporter` / `ImageExporter` classes as
`File → Export` — labels, scale bars, PiPs, rotated text, vector PDF
stamping, CMYK ICC conversion all behave identically.

### Platform plugin

To preserve text rendering parity with the GUI the CLI uses the
*native* Qt platform plugin on Windows and macOS (`windows` /
`cocoa`). The native plugin is the only one that initialises the OS
font database (DirectWrite on Windows, Core Text on macOS); the
`offscreen` plugin on Windows ships without a font directory and
renders every glyph as a filled rectangle (tofu). No window is ever
shown — `QPdfWriter` / `QImage` paint to a paint device, not a
window — so the native plugin behaves like a headless renderer in
practice.

On Linux the CLI defaults to `offscreen`, which uses fontconfig
(the system font db) and works without a `DISPLAY` (CI, SSH, Docker).

You can override the choice with `QT_QPA_PLATFORM` in the environment
if you need to (e.g. running under a Windows service account with no
GDI access — accept that text will tofu).

## Installation

The Windows installer (`ImageLayoutManager_Setup.exe`, produced by
`build_installer_windows.py`) ships `imagelayout-cli.exe` next to the
GUI exe. After install:

```powershell
"C:\Program Files\ImageLayoutManager\imagelayout-cli.exe" --help
```

Or use the **ImageLayoutManager CLI (shell)** Start Menu entry, which
opens `cmd.exe` in the install directory with `imagelayout-cli --help`
already executed — from there you can run any verb without typing the
full path.

To call `imagelayout-cli` from any shell, add the install directory to
`PATH` manually (System Properties → Environment Variables) — the
installer deliberately does **not** modify `PATH` to avoid surprising
existing user customisations.

## Building

Dev runs (no install needed):

```powershell
python cli_main.py inspect path\to\file.figpack
```

Standalone CLI binary (without the GUI installer):

```powershell
pyinstaller --noconfirm imagelayout-cli.spec
# → dist\imagelayout-cli\imagelayout-cli.exe
```

Combined GUI + CLI installer:

```powershell
python build_installer_windows.py
# → dist\ImageLayoutManager_Setup.exe   (contains both exes)
```

Both specs exclude Qt modules the app doesn't need (`QtNetwork`,
`QtMultimedia`, `QtWebEngine*`, `Qt3D*`, `QtQml`, `QtQuick*`, …) but
include matplotlib + numpy because `$...$` LaTeX math depends on them.
