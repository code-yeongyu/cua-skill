# Sandbox usage reference

Read this only when the user explicitly wants automation isolated from their real machine, or when an action is destructive enough that you proactively suggest a sandbox. For the default localhost flow, see [`localhost-usage.md`](localhost-usage.md).

## Sandbox runtimes

cua supports multiple sandbox backends. Pick the one whose dependencies the user already has:

| Runtime | Guest OS | Host requirement |
|---|---|---|
| **docker** | Linux (XFCE / Kasm container) | Docker Desktop or Colima |
| **qemu** | Linux full VM | QEMU installed |
| **lume** | macOS guest on macOS host | macOS host, [`lume`](https://github.com/trycua/lume) CLI installed |
| **tart** | macOS guest on Apple Silicon | [Tart](https://tart.run/) installed |

`docker` is the easiest first pick - most macOS / Linux developers already have it. `lume` and `tart` are needed only when the **guest** must be macOS.

For cloud-hosted sandboxes (`cua.ai`), see the cloud section at the bottom of this file.

## Lifecycle

### Start a sandbox

```bash
# minimal - cua picks a runtime, defaults to a Linux XFCE container
cua sandbox start

# explicit runtime + OS choice
cua sandbox start --runtime docker --os linux --kind container

# named sandbox so you can target it later
cua sandbox start --name dev-sb --runtime docker --os linux

# Linux VM via QEMU
cua sandbox start --runtime qemu --os linux --kind vm

# macOS guest via Lume (host must be macOS, lume CLI must be on PATH)
cua sandbox start --runtime lume --os macos
```

Flag names vary across cua versions. If the call fails with argparse errors, run `cua sandbox start --help` for the exact form your version accepts.

The command prints the assigned name on success; use that name as `<sandbox-name>` below.

### List active sandboxes

```bash
cua sandbox list
```

### Stop and destroy a sandbox

```bash
cua sandbox stop <sandbox-name>
```

Sandbox cleanup is **opt-in**, not automatic. If you started a sandbox during a task, stop it before ending the agent turn unless the user explicitly wants it left running.

## Targeting commands at a sandbox

Once a sandbox is up, route `cua do …` calls into it with the target flag. The exact flag name is usually `--target` or `--sandbox`; check `cua do <verb> --help` if uncertain.

```bash
# screenshot the sandbox display, not the host
cua do screenshot --target dev-sb --output /tmp/sb-1.png

# click inside the sandbox
cua do click 540 380 --target dev-sb

# run a shell command inside the sandbox
cua do shell "uname -a" --target dev-sb

# type into the focused sandbox window
cua do type "echo hello" --target dev-sb
cua do key --target dev-sb Return
```

When `--target` is omitted on a host with active sandboxes, behavior depends on the cua version - some versions default to the most recently started sandbox, others default to localhost. Always pass `--target` explicitly when sandboxes are in play.

## Why sandbox over localhost

Use a sandbox when:

- The automation runs untrusted code, opens unknown URLs, or installs random packages.
- The user wants to test something that would mutate `/Users/<you>` or `~/Documents` and you cannot easily roll back.
- The user is running on Linux/Windows and needs a **macOS** GUI to test something (Lume/Tart on macOS hosts).
- The user is on macOS and needs a clean **Linux** GUI without touching their host.

Do NOT default to a sandbox just to feel safe — sandboxes have startup cost (a Docker container needs 5-15s, a Lume VM 30-90s the first time), and `cua do screenshot` over a containerized display is heavier than capturing the host. Use a sandbox when isolation matters, localhost otherwise.

## Sandbox image options

Linux containers (the `--kind container` default for `--os linux --runtime docker`) ship with XFCE or Kasm preinstalled and an x11 server. You'll see a Linux desktop in screenshots. Common knobs:

```bash
# pin a specific image version
cua sandbox start --runtime docker --os linux --version 24.04

# full VM instead of container (heavier, but full kernel isolation)
cua sandbox start --runtime qemu --os linux --kind vm

# Windows 11 guest (via QEMU; requires significant RAM)
cua sandbox start --runtime qemu --os windows
```

Image catalog: see `cua image list` and `cua image --help` for what's actually available in your install.

## Persistence

By default sandboxes are **ephemeral** — `cua sandbox stop` destroys them and any files inside are lost. To keep state:

- Mount a host directory (cua docker runtime exposes this via a flag, often `--mount <host-path>:<guest-path>`; verify with `--help`).
- Inside the sandbox, write outputs to the mounted directory before stopping.

For long-lived development environments, prefer named sandboxes that you intentionally leave running and reconnect to with `cua do <verb> --target <name>`.

## Cloud sandboxes (cua.ai)

If the user has a `CUA_API_KEY` set and wants to run automation in the cloud rather than on their machine:

```bash
export CUA_API_KEY=sk_cua-...
cua sandbox start --cloud --os linux --region north-america
```

Cloud sandboxes are the same surface but execution is remote. Cost applies — the user pays per minute of running VM. Always `cua sandbox stop <name>` when done.

Refer to <https://docs.trycua.com> for the current region list and pricing.

## When sandbox is the wrong answer

- If the user only wants to read their local files, sandboxing adds overhead with zero benefit. Use localhost shell or pi's built-in bash.
- If the user wants to control their personal browser session (logged-in cookies, saved passwords), a sandbox can't see them. Use localhost.
- If the user wants to delegate "just do this multi-step task" without specifying isolation, default to localhost and ask about sandboxing only if the task looks risky.

## Cleanup hygiene

End every sandbox-using turn with `cua sandbox stop <name>` (or `cua sandbox list` to verify nothing is leaking) unless the user explicitly asked to keep it. Leaked sandboxes consume host RAM/CPU and, on cloud mode, the user's money.
