# Installation reference

Setting up cua so the `cua` CLI is on PATH and Python imports resolve cleanly.

## TL;DR for senpi users

```bash
# 1. create an isolated venv (Python 3.12 recommended)
uv venv --python 3.12 ~/.senpi/.pi/cua-venv

# 2. install cua + its sibling packages
uv pip install --python ~/.senpi/.pi/cua-venv/bin/python cua

# 3. expose the cua binary on PATH (pick one)
export PATH="$HOME/.senpi/.pi/cua-venv/bin:$PATH"   # quick, ad-hoc
# or symlink it permanently
ln -sf ~/.senpi/.pi/cua-venv/bin/cua ~/.local/bin/cua

# 4. one-time host consent (macOS will prompt for Accessibility + Screen Recording)
cua do-host-consent

# 5. smoke test
cua do screenshot --output /tmp/cua-test.png && ls -lh /tmp/cua-test.png
```

If `cua do screenshot` produces a 0-byte file or a black image, the controlling terminal is missing Screen Recording permission. Fix it in macOS **System Settings → Privacy & Security → Screen Recording** and toggle the entry for whichever terminal (iTerm2, Ghostty, WezTerm, VS Code, etc.) launched the cua process.

## Why a venv and not system pip

Homebrew Python (the default `python3` on macOS for users on Apple Silicon with `/opt/homebrew/bin/python3`) ships with [PEP 668](https://peps.python.org/pep-0668/) `EXTERNALLY-MANAGED`. Running `pip install cua` against it fails with:

```
error: externally-managed-environment
× This environment is externally managed
```

The clean fix is a dedicated venv. Don't reach for `--break-system-packages` - it silently risks breaking your Homebrew Python.

## Packages that `pip install cua` actually pulls

The `cua` distribution on PyPI is a **meta-package**. After install, the venv contains:

| Package | Role |
|---|---|
| `cua` | meta - lazy attribute re-export from the others |
| `cua-agent` | the `ComputerAgent` autonomous loop |
| `cua-sandbox` | sandbox + Localhost classes |
| `cua-computer` | computer-use interface framework |
| `cua-cli` | the `cua` binary you actually call |
| `cua-auto` | pynput-based mouse/keyboard/screen backend |
| `cua-core` | shared primitives |

You can confirm with `uv pip list --python <venv-python>` or, inside the venv, `pip list`.

## Known issue: `ModuleNotFoundError: No module named 'agent'`

On cua `0.1.6`, the meta-package's `__init__.py` lazy-imports `ComputerAgent` from a module named `agent`, but the installed distribution exposes it as `cua_agent`. As a result:

```python
from cua import ComputerAgent   # raises ModuleNotFoundError: No module named 'agent'
```

This is a packaging bug in cua itself, not an installation problem. Workarounds in order of preference:

1. **Use the CLI** — `cua do task "..."` doesn't trip the broken re-export, because the CLI imports `cua_agent` directly. Prefer this for any agent delegation.
2. **Import `cua_agent` directly in Python** if you must script:
   ```python
   from cua_agent import ComputerAgent  # works
   ```
3. **Watch [trycua/cua](https://github.com/trycua/cua) for a release > 0.1.6**, which will fix the lazy attribute table.

The sibling Sandbox / Image / Localhost imports through `cua` work fine; only `ComputerAgent` is affected.

## Optional: `cua[omni]` (AGPL-3.0)

`cua[omni]` adds `cua-som` (Set-of-Mark visual grounding via OmniParser + YOLO via Ultralytics). **`cua-som` is AGPL-3.0-or-later** because `ultralytics` is AGPL. The base `cua` install does NOT include it; you opt in explicitly:

```bash
uv pip install --python ~/.senpi/.pi/cua-venv/bin/python 'cua[omni]'
```

Activation is also opt-in at the agent level — `ComputerAgent` only loads SoM when you pass a model string prefixed with `omniparser+` or `omni+`. Standard models (Claude, GPT, Gemini) never trigger it.

If you don't want any AGPL code in the user's environment, do not install `cua[omni]` and do not use `omniparser+...` / `omni+...` model identifiers. The base `cua` install is fully MIT.

## macOS Accessibility + Screen Recording

`cua do click` and `cua do type` use pynput, which routes through the macOS Accessibility API. `cua do screenshot` uses CoreGraphics screen capture. Both require their own toggle:

- **System Settings → Privacy & Security → Accessibility** → enable for the terminal/IDE that launches `cua`.
- **System Settings → Privacy & Security → Screen Recording** → enable for the same.

The first time you run `cua do-host-consent`, macOS will surface both prompts. If you skip them or revoke them later, all subsequent calls silently fail.

If you switch terminals (e.g., from iTerm to Ghostty), repeat the toggles for the new app. Permission is per-binary, not per-user.

## Linux / Windows installation

The CLI install path is identical (`uv venv` + `uv pip install cua`). Permissions differ:

- **Linux X11**: pynput uses Xlib, no extra config. Make sure `DISPLAY` is set.
- **Linux Wayland**: pynput's Wayland support is partial - many key-press operations work through `ydotool` or `wtype` if you install them and configure pynput to use them. If `cua do key` fails on Wayland, see [`troubleshooting.md`](troubleshooting.md).
- **Windows**: pynput uses SendInput through ctypes, no extra permission grants needed. Run `cua do-host-consent` for the consent log, but no OS prompt appears.

## Configuring pi/senpi to use the venv

Once installed, point pi at the venv Python in `~/.pi/cua.json` or `~/.senpi/.pi/cua.json` (if you also use senpi):

```json
{
  "python": {
    "executable": "/Users/<you>/.senpi/.pi/cua-venv/bin/python"
  }
}
```

This is only relevant if you're using the legacy `pi-cua-integration` extension that boots a Python daemon. With this skill, you only need `cua` on PATH — no daemon, no config-driven Python executable.

## Uninstall

```bash
rm -rf ~/.senpi/.pi/cua-venv
unlink ~/.local/bin/cua    # if you symlinked it
```

That's all — cua doesn't write to system locations.
