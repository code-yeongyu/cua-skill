# ComputerAgent via `cua do task`

Read this when the user describes a **goal** rather than a sequence of actions. Examples: "open Firefox, go to cua.ai, take a screenshot of the homepage", "fill in this form with my saved details", "find the cheapest flight to Tokyo next month and screenshot the result".

`cua do task` hands the goal to cua's `ComputerAgent`, which runs an internal vision+action loop with its own LLM until the goal is met or it hits the turn cap. You wait for the trajectory and surface the final result.

## When ComputerAgent earns its place

Compared to scripting a click/type sequence yourself:

- **Saves your context budget.** The screenshot-decide-act loop runs inside the sub-agent; you only see the summary, action count, and final text. Manual scripting would put every intermediate screenshot in your conversation.
- **Tolerates UI drift.** If a button moves or the layout shifts, the sub-agent re-decides; a hand-coded click sequence breaks.
- **Worth the extra LLM cost** when there are more than ~5 steps, or when the steps are visually obvious but textually awkward to describe.

When NOT to use it:

- One-off simple actions (single click, single screenshot). Script it directly.
- Privacy-sensitive flows (the sub-agent's LLM provider sees screenshots).
- Anything where you need to inspect every intermediate state for debugging.

## Basic invocation

```bash
cua do task "Open Firefox and navigate to https://cua.ai, then capture the homepage"
```

The CLI prints progress lines as the sub-agent works, then a final summary block with the action count, screenshot count, and the agent's closing message.

## Choosing a model

`ComputerAgent` uses [LiteLLM](https://docs.litellm.ai/) under the hood, so any provider+model that supports tool-use + vision works:

```bash
cua do task "..." --model anthropic/claude-sonnet-4-5
cua do task "..." --model openai/gpt-5
cua do task "..." --model gemini/gemini-2.5-pro
```

Provider API keys are read from the standard env vars: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`. Set them in the shell that runs `cua` before issuing the task.

The default model depends on cua version - check `cua do task --help`. Typical defaults are `anthropic/claude-sonnet-4-5` or whatever cua currently considers the most capable computer-use-ready model.

## Capping cost - `--max-turns`

Each turn includes a screenshot, so trajectory cost is roughly linear in turn count. Use `--max-turns` to bound it:

```bash
cua do task "Fill in the contact form with my saved details" --max-turns 10
```

`--max-turns` defaults are generous (often 30-50). For exploratory or non-critical tasks, 10-15 is usually enough; if the agent hits the cap, it returns partial progress and you can decide whether to extend.

## Targeting localhost vs sandbox

Same `--target` semantics as the rest of the cua CLI:

```bash
# delegate task on the user's real machine
cua do task "Open System Settings and turn on Do Not Disturb"

# delegate task inside a running sandbox
cua sandbox start --name web-sb --runtime docker --os linux
cua do task "Install firefox and open it" --target web-sb
```

For destructive or experimental goals, **prefer sandbox**. The agent makes mistakes and a mis-click in localhost mode can do real damage.

## What you get back

The CLI emits a structured summary at the end:

```
Task complete (12 actions, 5 screenshots).
Final response: I successfully opened Firefox and captured the homepage. ...
```

Programmatic access to the full trajectory:

```bash
cua trajectory list                       # see recorded trajectories
cua trajectory show <trajectory-id>       # full action sequence + screenshots
```

Trajectories include every screenshot and every action the sub-agent took. Useful for debugging "why did it do that?" or replaying a flow.

## Common patterns

### Goal-mode exploration

```bash
cua do task "Find the current top story on Hacker News and read me the title and a one-sentence summary" --max-turns 8
```

The agent will open a browser, navigate to news.ycombinator.com, screenshot, parse, and respond.

### Sandboxed risky automation

```bash
cua sandbox start --name install-test --runtime docker --os linux
cua do task "Install nodejs and run 'node --version' to confirm" --target install-test --max-turns 15
cua sandbox stop install-test
```

### Form-filling with confirmation

When the user wants you to fill but not submit:

```bash
cua do task "Fill in the registration form on this page with name 'Test User', email 'test@example.com', but DO NOT click Submit. Stop at the filled form." --max-turns 10
```

Note: the agent honors negative constraints in the goal text, but verify with a final screenshot. If the goal looks ambiguous, ask the user once before delegating.

## Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'agent'` if using the Python API | cua 0.1.6 lazy-import bug | Use the CLI (`cua do task`), or `from cua_agent import ComputerAgent` directly. See [`troubleshooting.md`](troubleshooting.md). |
| Agent loops without progress until `--max-turns` exhausted | Goal too vague, or page state is unusual | Tighten the goal text; provide a "stop condition" sentence; consider scripting manually instead. |
| Agent clicks the wrong UI element | Vision model misread the screenshot | Reduce display scale (smaller resolution = fewer pixels to interpret), or switch to a stronger model with `--model`. |
| `RateLimitError` mid-trajectory | LLM provider throttled | Wait and retry. Consider a model with higher quotas. |
| Empty `final response` | Agent finished but couldn't summarize | Trajectory still has the actions - check with `cua trajectory show`. |

## Cost awareness

A 10-turn trajectory with Claude Sonnet on a 1080p screenshot is ~30k-60k input tokens (mostly image). At current pricing, that's $0.10-$0.20 per task. For dozens of trajectories, costs add up — set `--max-turns` low when iterating on prompts, and stop sandboxes promptly to avoid leaving them running while you think.

## When `cua do task` is the wrong answer

- The goal is one click + one screenshot. Just script it.
- The user is debugging an automation and wants to see every step. Script it.
- The user wants pixel-precise control or specific timing. Script it.
- The user doesn't want their screenshots leaving their machine. Use sandbox + local-only model, or script it manually.
