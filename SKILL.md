---
name: cua-skill
description: "MUST USE whenever the user wants to automate a real desktop or sandbox - clicking, typing, scrolling, screenshotting, running an OS shell command, or handing a high-level 'open browser and do X' task to an autonomous computer-use agent. Wraps the trycua/cua Python toolkit via its `cua` CLI - pynput-based, cross-platform (macOS / Linux / Windows). NO custom tools are registered; you call `cua` through pi's built-in bash and read screenshots back through the Read tool. Triggers: cua, computer use, computer-use, GUI automation, screenshot the desktop, click on the screen, type into the active app, scroll the page, control my computer, drive my browser, sandbox, docker sandbox, QEMU sandbox, Lume sandbox, ComputerAgent, cua do, cua sandbox, 컴퓨터 자동화, 스크린샷 찍어, 내 컴퓨터 조작, 브라우저 열어서, 샌드박스, 화면 자동화, 마우스로 클릭, 키보드 타이핑, computer use 위임, 자동으로 클릭, 자율 에이전트로 처리."
---

# cua-skill

`cua` (from [trycua/cua](https://github.com/trycua/cua)) is a Python-native computer-use automation framework. The `cua` CLI is a single binary that drives the mouse, keyboard, screen, shell, and an autonomous `ComputerAgent` loop. This skill teaches you how to invoke it directly from pi's built-in bash - **no extension tools are registered**, so every action is shell-mediated.

## When to reach for cua

Use cua whenever the user wants the agent to operate a GUI as a human would, or to delegate a multi-step "do this in the browser / in this app" task. Specifically:

- Take a screenshot of the user's desktop (or a sandbox display).
- Click, double-click, drag, type text, press key chords, scroll at coordinates.
- Run a shell command on the host or inside a sandbox.
- Spin up an isolated Docker / QEMU / Lume / Tart sandbox to run untrusted automation.
- Hand a natural-language goal ("open Firefox, search for X, click the first result") to cua's `ComputerAgent` and wait for the trajectory.

If the user only wants to read a file, run a normal CLI, or write code, **this skill does not apply** - use pi's existing read/bash/edit tools.

## Operating modes

cua has three execution targets. Decide which one applies before issuing commands; the surface is mostly identical but the safety implications are not.

| Mode | Where actions actually run | Sandboxed? | When to pick it |
|---|---|---|---|
| **localhost** (default) | the user's real machine | no | The user explicitly wants their actual desktop driven. macOS, Linux, and Windows all work via pynput. |
| **sandbox** | a Docker container or VM (QEMU / Lume / Tart) | yes | The user wants risky automation isolated, or wants a clean Linux/Windows display. Requires `cua sandbox start` first. |
| **cloud** | a cua.ai-hosted VM | yes | The user has `CUA_API_KEY` and wants ephemeral cloud execution. |

If the user did not specify and the action is destructive (rm-anything, writing to system locations, irreversible UI clicks), pause and confirm: "Run on your real machine, or in a fresh sandbox?"

## First-time host consent

Localhost mode needs OS-level input + screen-recording permission. The very first time `cua` touches the host, run:

```bash
cua do-host-consent
```

This is a one-shot grant. macOS will prompt for Accessibility + Screen Recording permission on the controlling terminal/IDE. After approval, subsequent `cua do …` calls work silently. If `cua do click` silently does nothing or `cua do screenshot` produces a black image, the user hasn't granted permission yet.

Full installation walkthrough: [`references/installation.md`](references/installation.md).

## Core surface — `cua do <verb>`

Every desktop action goes through `cua do`. The verb taxonomy:

| Action | Command shape |
|---|---|
| Click | `cua do click <x> <y>` |
| Type text | `cua do type "<text>"` |
| Key chord | `cua do key <chord>` (e.g. `cmd+s`, `Return`, `ctrl+shift+t`) |
| Scroll | `cua do scroll <x> <y> --dy <amount>` |
| Screenshot | `cua do screenshot --output /tmp/cua-<unix-ts>.png` |
| Shell exec | `cua do shell "<command>"` (use pi's own bash on localhost — see note below) |
| Agent task | `cua do task "<natural-language goal>" [--model <id>] [--max-turns <N>]` |

Flag names sometimes drift across cua minor versions; if the first call fails with an argparse error, fall back to `cua do <verb> --help` immediately, capture the actual flag set, and continue. Don't guess.

Concrete recipes for each verb: [`references/localhost-usage.md`](references/localhost-usage.md).

## The screenshot → Read pattern

cua writes screenshots to disk; you ingest them into context by **calling pi's Read tool on that path**. Do NOT try to base64-encode the file yourself - Read already attaches PNG/JPEG payloads as image content blocks.

```bash
# 1. capture
TS=$(date +%s%N)
cua do screenshot --output "/tmp/cua-${TS}.png"
# 2. then in the same turn, call pi's Read tool with absolute path /tmp/cua-${TS}.png
```

This is the entire reason cua is worth a skill rather than a tool - the Read tool already does image-inline injection, so we get the same vision capability with zero custom plumbing.

## Shell on localhost - prefer pi's bash

On localhost mode, `cua do shell "ls"` and pi's built-in bash run in the **same shell environment**. Use pi's bash directly unless you specifically need cua's per-call timeout semantics or the same syntax to also work when the user switches to a sandbox. The `cua do shell` form really earns its place only on sandbox / cloud modes, where it routes the command into the isolated environment.

## Reference files - load only what you need

Each file in `references/` is independently consumable. Read only the one that matches the current request to keep context lean.

| File | Read when |
|---|---|
| [`references/installation.md`](references/installation.md) | Setting up cua for the first time, fixing a missing-dependency error, configuring the venv that pi will exec, or diagnosing PEP 668 issues on Homebrew Python. |
| [`references/localhost-usage.md`](references/localhost-usage.md) | Driving the user's real machine — mouse/keyboard/screenshot/shell recipes with exact flag forms. |
| [`references/sandbox-usage.md`](references/sandbox-usage.md) | Starting / stopping Docker / QEMU / Lume / Tart sandboxes, targeting commands at a specific sandbox, capturing sandbox-side screenshots. |
| [`references/computer-agent.md`](references/computer-agent.md) | Delegating a multi-step goal to `ComputerAgent` via `cua do task`, picking a model, reading trajectory output, capping cost with `--max-turns`. |
| [`references/troubleshooting.md`](references/troubleshooting.md) | Common errors — `ModuleNotFoundError: No module named 'agent'`, permission-denied on macOS, pynput backend issues on Wayland, runaway clicks, etc. |

## Critical conventions

Some defaults are non-obvious and have severe consequences when violated, so they're worth stating in the body rather than buried in a reference file.

### Never auto-drive a destructive GUI

Mouse and keyboard on localhost is privileged: a single misclick can confirm a destructive dialog, send a message, or fill a payment form. If the user said "automate something" but did not specify the exact target, stop and confirm before driving the real GUI. A short screenshot-and-confirm exchange is much cheaper than an unrecoverable click.

### Logical points, not physical pixels

`cua do click 100 100` operates in **logical points**, the same coordinate system you see in macOS System Settings or in browser devtools. `cua_auto` already reads `NSScreen.backingScaleFactor()` (macOS), `GDK_SCALE` / `QT_SCALE_FACTOR` (Linux), and the Windows DPI scaling factor, and converts internally. Do not pre-multiply by 2 for Retina displays.

### One screenshot per decision, not per turn

Screenshots are large image tokens. Take one, decide on a sequence of actions from it, execute them, then re-screenshot only if the next decision actually depends on the visual state. Re-screenshotting after every click is wasteful and slow.

### When the user names a sandbox or mode, respect it

If the user said "in a sandbox" or "in a Docker", do not collapse to localhost just because it's the default. Start the sandbox first, target it explicitly, and only fall back to localhost with the user's confirmation. The opposite is also true: if they said "on my actual machine", do not spawn a sandbox.

## Validation reminder

After editing this skill or any reference file under `references/`, run the senpi/Claude Skills validator before declaring work done:

```bash
python3 ~/.agents/skills/skill-creator/validate-skills.py ~/.agents/skills/cua-skill
```

Fix every violation until the validator prints `OK: 1 skill(s) valid`. Silently-rejected skills are worse than no skill.
