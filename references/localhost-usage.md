# Localhost usage reference

How to drive the user's real machine via `cua do …`. macOS, Linux, and Windows share the same surface; differences are noted inline.

Prerequisites: [`installation.md`](installation.md) completed, `cua do-host-consent` run once, OS-level permissions granted.

## Verbs at a glance

| Verb | Purpose | Typical call |
|---|---|---|
| `click` | Single click at (x,y) | `cua do click 540 380` |
| `double-click` | Double click | `cua do double-click 540 380` |
| `right-click` | Secondary click | `cua do right-click 540 380` |
| `move` | Move cursor without clicking | `cua do move 540 380` |
| `type` | Type literal text | `cua do type "Hello, world"` |
| `key` | Press a key chord | `cua do key cmd+s` |
| `scroll` | Scroll at coordinates | `cua do scroll 540 380 --dy -3` |
| `screenshot` | Capture the active display | `cua do screenshot --output /tmp/cua-1.png` |
| `shell` | Execute a shell command | `cua do shell "ls /Users"` |
| `task` | Delegate a goal to ComputerAgent | `cua do task "open Firefox"` (see `computer-agent.md`) |

If a verb name above doesn't exist in your cua version, run `cua do --help` and `cua do <verb> --help` - the verb table evolves between minor releases.

## The screenshot-then-Read recipe

The single most important recipe. cua writes the PNG to disk; pi's Read tool ingests it as an image content block.

```bash
# capture with a unique filename so concurrent calls don't collide
TS=$(date +%s%N)
SHOT="/tmp/cua-${TS}.png"
cua do screenshot --output "${SHOT}"
```

Then in the **same agent turn**, call pi's Read tool with the absolute path `/tmp/cua-<ts>.png`. The PNG is attached to the next assistant message as inline image content. Do not base64 it manually, do not pipe it through stdin — Read already does the right thing.

If the user wants the full screen including multiple displays, check `cua do screenshot --help` for `--display <id>` or equivalent.

## Coordinate system - logical points

`cua do click 100 100` operates in **logical points**, the coordinate system you see in System Settings or browser devtools. Internally `cua_auto.screen.get_display_scale()` reads:

- macOS: `AppKit.NSScreen.mainScreen().backingScaleFactor()` (usually 2.0 on Retina, 1.0 on external 1080p).
- Windows: `ctypes.windll.shcore.GetScaleFactorForDevice(0) / 100.0`.
- Linux: `GDK_SCALE` or `QT_SCALE_FACTOR` env var.

Don't pre-multiply. If a screenshot shows a button at (1080, 720), click `cua do click 1080 720` — `cua_auto` handles the scale conversion when sending the OS-level event.

## Click family

```bash
# single left click
cua do click 540 380

# double left click
cua do double-click 540 380

# right click for context menu
cua do right-click 540 380

# move cursor without clicking (useful to hover something to reveal a tooltip)
cua do move 540 380
```

If the click "doesn't seem to do anything", common causes:

1. Window is not focused. Use `cua do click <x> <y>` on the title bar first.
2. The click landed on a different display. Specify `--display` if cua's default is wrong.
3. Accessibility permission revoked. Re-grant in System Settings.

## Type and key

```bash
# literal text (shell-quote carefully; cua passes the string to pynput.keyboard.type())
cua do type "Hello, world!"

# multiline - real newlines are typed as Enter presses
cua do type "line one
line two"

# special characters: emoji, non-ASCII, accents all work because cua_auto.keyboard uses
# the platform's IME-aware text input rather than per-character key events.
cua do type "안녕하세요 🌏"
```

```bash
# key chords - join with '+'
cua do key cmd+s          # macOS save
cua do key ctrl+s         # Linux/Windows save
cua do key Return         # bare named keys also work
cua do key cmd+shift+t    # multi-modifier chord

# multiple separate presses in sequence
cua do key Escape
cua do key Tab
cua do key Tab
cua do key Return
```

The key-name vocabulary follows pynput's: `cmd` (macOS) / `ctrl`, `alt`, `shift`, `option`, `Return`, `Tab`, `Escape`, `Backspace`, `Up`, `Down`, `Left`, `Right`, `Home`, `End`, `Page_Up`, `Page_Down`, `F1`…`F19`. Letters and digits are themselves: `cua do key a`, `cua do key 5`.

When in doubt, prefer `cua do type "X"` for literal characters and `cua do key X+Y` for chords. Don't mix them.

## Scroll

```bash
# scroll down 3 ticks at (540, 380)
cua do scroll 540 380 --dy -3

# scroll up 5 ticks
cua do scroll 540 380 --dy 5

# horizontal scroll (rare, but supported on trackpads)
cua do scroll 540 380 --dx 2
```

A "tick" is one notch on a physical scroll wheel; on a smooth trackpad it's roughly 1/8 of a viewport. Empirically, `--dy -3` to `--dy -5` is a reasonable "page down".

## Shell

On localhost, prefer pi's built-in bash tool — it has the same shell environment and you get pi's standard output capture. Use `cua do shell` only when you specifically want cua's per-call timeout or you want a single call that also works on sandbox/cloud modes by adding `--target <sandbox-name>`.

```bash
cua do shell "ls -la /Users"
cua do shell "open -a Safari"          # macOS app launch
cua do shell "powershell.exe -c '...'" # Windows shell from cua on a Windows host
```

If the command might run for a long time, pass `--timeout-ms`:

```bash
cua do shell "sleep 5 && echo done" --timeout-ms 10000
```

## Common end-to-end patterns

### Find a button on screen, then click it

```bash
# 1. screenshot
TS=$(date +%s%N) ; SHOT="/tmp/cua-${TS}.png"
cua do screenshot --output "${SHOT}"
# 2. (agent turn) Read /tmp/cua-${TS}.png to look at it and decide coordinates
# 3. click
cua do click 540 380
# 4. verify
TS=$(date +%s%N) ; SHOT="/tmp/cua-${TS}.png"
cua do screenshot --output "${SHOT}"
# 5. Read again to confirm the click had the intended effect
```

### Fill a login form

```bash
cua do click 720 320         # focus the email field
cua do type "user@example.com"
cua do key Tab
cua do type "<password>"
cua do key Return
```

If `cua do key Tab` doesn't move focus the way you expect, the page may use a non-standard tab order — fall back to clicking each field explicitly.

### Open an app then drive it

```bash
# macOS
cua do shell "open -a 'Visual Studio Code'"
sleep 1   # give the app a moment to focus
cua do key cmd+shift+p
cua do type "Reload Window"
cua do key Return
```

## Concurrency and serialization

`cua do …` calls are serialized at the OS-event level on the same host — you can't reliably interleave clicks and keystrokes from parallel `cua do` invocations and expect deterministic ordering. Keep automation sequential within a single agent turn, or insert short `sleep` calls between independent actions when timing matters.

## When localhost is the wrong answer

If the user said "in a sandbox" or "in Docker" or "in a fresh VM", do not localhost. Switch to [`sandbox-usage.md`](sandbox-usage.md). If the user wants to delegate the whole flow ("just open Firefox and search X for me"), prefer [`computer-agent.md`](computer-agent.md) over manual click sequences.
