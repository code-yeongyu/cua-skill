# Troubleshooting

Specific errors and their fixes. Most cua failures come from OS-permission gaps, the 0.1.6 packaging bug, or a stale sandbox.

## `ModuleNotFoundError: No module named 'agent'`

**Where it happens:** Anywhere that imports `ComputerAgent` through the `cua` meta-package, including:

```python
from cua import ComputerAgent     # raises
```

**Cause:** cua 0.1.6's `cua/__init__.py` lazy-attribute table maps `ComputerAgent` to a module literally named `agent`, but the installed distribution exposes it as `cua_agent`. This is a packaging bug in cua, not in your install.

**Fixes**, in order of preference:

1. **Use the CLI.** `cua do task "…"` reaches `cua_agent` directly, bypassing the broken meta re-export. Almost every workflow this skill describes uses the CLI, so you rarely hit this in practice.
2. **Direct Python import** when scripting:
   ```python
   from cua_agent import ComputerAgent   # works
   ```
3. **Wait for cua > 0.1.6.** Watch the [trycua/cua releases](https://github.com/trycua/cua/releases) page.

`Sandbox`, `Image`, and `Localhost` imports through the meta-package are unaffected — only `ComputerAgent` triggers the bug.

## `cua do click` silently does nothing on macOS

**Cause:** the terminal/IDE that launched `cua` lacks Accessibility permission.

**Fix:**

1. Open **System Settings → Privacy & Security → Accessibility**.
2. Find the entry for your terminal (iTerm2, Ghostty, WezTerm, Apple Terminal, VS Code, etc.) — the binary that invoked `cua`.
3. Toggle it ON. If the entry doesn't exist, click `+` and add the app.
4. Restart the terminal (some apps cache the permission state at launch).
5. Re-run `cua do click 540 380`.

Permission is per-binary. If you switch from iTerm to Ghostty, you must grant Ghostty too. The same applies to "swap your editor" - VS Code's integrated terminal needs its own grant.

## `cua do screenshot` returns a black or 0-byte image

**Cause:** missing Screen Recording permission.

**Fix:** same path as above but **System Settings → Privacy & Security → Screen Recording**. Toggle the controlling terminal/IDE ON. Restart. Re-test.

If the screenshot is non-black but shows the wrong display, pass `--display <n>` (run `cua do screenshot --help` for the flag name in your cua version).

## `cua do key cmd+s` does nothing on Linux Wayland

**Cause:** pynput's Wayland support is limited; many key events require a compositor-side helper.

**Fix options:**

1. Switch the session to X11 (login screen → gear icon → "Ubuntu on Xorg" or equivalent). Wayland support in pynput is a known weak spot.
2. Install `ydotool` and configure pynput's backend env var (varies; check pynput's Wayland docs).
3. As a last resort, route the keystroke through `xdotool` (X11) or `ydotool` (Wayland) directly via `cua do shell "xdotool key cmd+s"` — but this bypasses cua's coordinate translation, so use sparingly.

## `error: externally-managed-environment` on `pip install cua`

**Cause:** Homebrew Python on macOS enforces PEP 668; system pip refuses to install into Homebrew's Python directly.

**Fix:** create a dedicated venv (recommended) - see [`installation.md`](installation.md) for the `uv venv` recipe. Do not use `--break-system-packages` — it can silently break Homebrew's Python.

## `cua: command not found`

**Cause:** the venv's `bin/` isn't on PATH.

**Fix:**

```bash
# either prepend the venv to PATH (transient)
export PATH="$HOME/.senpi/.pi/cua-venv/bin:$PATH"

# or symlink permanently to a directory already on PATH
ln -sf ~/.senpi/.pi/cua-venv/bin/cua ~/.local/bin/cua

# verify
which cua && cua --version
```

If your shell init (`.zshrc`, `.bashrc`) is responsible, add the export there so future shells inherit it.

## `cua sandbox start` fails on Docker runtime

**Symptoms:** "Cannot connect to the Docker daemon", "no such image", "permission denied".

**Fixes by symptom:**

- **"Cannot connect to the Docker daemon"** → Docker Desktop or Colima isn't running. Start it; `docker info` should succeed before retrying.
- **"no such image"** → run `cua image list` to see what's available; pull the missing image via `cua image pull <name>` if your version supports it, otherwise upgrade cua.
- **"permission denied" on Linux** → add your user to the `docker` group: `sudo usermod -aG docker $USER`, then log out + back in.

## QEMU runtime is very slow on Apple Silicon

QEMU runs guest x86_64 code under emulation on M-series macs - it's an order of magnitude slower than a native Linux container. If you don't need full-kernel isolation, switch to `--runtime docker --kind container` for the same Linux surface at native speed.

For macOS guests on Apple Silicon, use Lume or Tart instead of QEMU.

## Sandbox left running after the agent turn ends

**Cause:** the agent forgot to call `cua sandbox stop` before exiting.

**Cleanup:**

```bash
cua sandbox list                # see what's running
cua sandbox stop <name>         # stop a specific one
# or stop all, if your cua version supports it
cua sandbox stop --all
```

For cloud sandboxes, also confirm they're stopped in the [cua.ai dashboard](https://cua.ai) — leaked cloud VMs cost real money.

## `cua do task` hits `--max-turns` without finishing

**Cause:** goal too vague, or the page state is unusual enough that the sub-agent's vision model keeps re-deciding.

**Fixes:**

1. **Tighten the goal text** with a clear stop condition. Bad: "do my taxes". Better: "open the IRS Free File homepage and screenshot the list of providers; stop there".
2. **Raise `--max-turns`** for genuinely long tasks. Defaults are 30-50; bumping to 100 is fine for tasks that need it.
3. **Switch model** — a stronger model finishes the same task in fewer turns. `--model anthropic/claude-opus-4-7` (if available in your LiteLLM config) usually beats Sonnet on visual tasks at the cost of higher per-turn price.
4. **Decompose** — split the task into sub-tasks and run each separately. Cheaper and easier to debug.

## `cua trajectory show <id>` is empty

**Cause:** trajectory recording is opt-in in some cua versions, or the trajectory was rotated out.

**Fix:** check `cua trajectory list` for the actual recorded IDs. If recording isn't on, see `cua trajectory --help` for the enable flag in your version.

## Pynput backend errors mentioning Quartz / CGEvent

**Cause:** the macOS Quartz framework binding is in a weird state (often after an OS update).

**Fix:**

```bash
# reinstall pynput inside the venv
uv pip install --python ~/.senpi/.pi/cua-venv/bin/python --force-reinstall pynput
# also ensure the Python's `pyobjc` is current
uv pip install --python ~/.senpi/.pi/cua-venv/bin/python --upgrade pyobjc-framework-Quartz
```

If errors persist, recreate the venv from scratch (see [`installation.md`](installation.md)).

## "It works in iTerm but not in tmux inside iTerm"

**Cause:** tmux/screen multiplexers sometimes hide the inheriting process from macOS's permission scope.

**Fix:** grant Accessibility / Screen Recording permission to **tmux's** binary, not just iTerm's. If `which tmux` returns `/opt/homebrew/bin/tmux`, that path needs the toggle. Same idea for `screen`.

## "`cua` works at the CLI but pi-cua-integration daemon says cuaAvailable: false"

The Python the daemon runs is not the venv Python.

**Fix:** set `python.executable` in `~/.pi/cua.json` (or `~/.senpi/.pi/cua.json`):

```json
{
  "python": {
    "executable": "/Users/<you>/.senpi/.pi/cua-venv/bin/python"
  }
}
```

Restart the pi/senpi session so the new daemon picks up the path. The new daemon's `ready` event should now show `cuaAvailable: true`.

This is only relevant if you're using the legacy `pi-cua-integration` extension. With this skill (CLI-only, no daemon), the venv just needs `cua` on PATH.

## Still stuck

1. Capture the full error output (stderr + stdout).
2. Check `cua --version` and the `cua` install path (`which cua`).
3. Run `cua do --help` and the failing verb's `--help` — flag names occasionally change between versions.
4. Search [trycua/cua issues](https://github.com/trycua/cua/issues) before filing — the lazy-import bug, Wayland gaps, and Docker permission gotchas are all known.
