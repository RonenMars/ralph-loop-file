# ralph-loop-file

A Claude Code plugin that provides `/ralph-loop-file` — a sibling of the upstream [`ralph-loop`](https://github.com/anthropics/claude-code-plugins/tree/main/ralph-loop) that reads its prompt from a **file** instead of positional shell arguments.

Use this whenever your prompt contains anything the shell would try to interpret: apostrophes, backticks, dollar signs, newlines, or any of the other punctuation that turns multi-line task descriptions into syntax errors.

## Why a separate plugin (not a fork)

The upstream `ralph-loop` slash command works perfectly for short, single-line, shell-safe prompts. It accepts the prompt positionally:

```text
/ralph-loop Build a todo API --completion-promise 'DONE'
```

That breaks the moment the prompt contains:

- Apostrophes — `don't`, `can't`
- Backticks — `` `useEffect` ``, `` `myFunction()` ``
- Dollar signs — `$variable`, `${PATH}`
- Newlines — multi-paragraph task descriptions
- Any other unbalanced shell-special character

Symptom: zsh fails the tool call with `(eval): unmatched '` (or similar). The shell tokenizes the entire prompt before Claude Code ever sees it, and chokes on the first unbalanced quote.

`/ralph-loop-file` sidesteps this by injecting a `--prompt-file` flag and letting the setup script read the prompt with `cat` **after** the shell has finished tokenizing flags. Prompt content never touches shell tokenization.

**Why a sibling instead of patching upstream:** the upstream `ralph-loop` plugin ships in `anthropics/claude-plugins-official`, cloned into `~/.claude/plugins/marketplaces/claude-plugins-official/`. That directory is overwritten by Claude Code's plugin update — any local patch is wiped on the next sync. Forking adds rebase burden against Anthropic's main branch. A sibling plugin is cheaper and durable: it ships from its own repo, stacks alongside the original (which keeps working unchanged), and only carries the one diff that matters.

## Install

In Claude Code:

```text
/plugin marketplace add RonenMars/claude-marketplace
/plugin install ralph-loop-file@ronenmars
```

The plugin will be loaded immediately. Verify with `/plugin` (look under the `ronenmars` marketplace).

## Dependencies

| Tool | When needed | What happens if missing |
|---|---|---|
| `ralph-loop` (upstream plugin) | Always — `/ralph-loop-file` reuses the upstream Stop hook to drive the loop | Loop won't iterate. Install via `/plugin install ralph-loop@claude-plugins-official` |
| `bash` | Always — setup script | Hard error (you'd have bigger problems) |
| `cat` | Always — reads the prompt file | Hard error (same as above) |

`/ralph-loop-file` is intentionally a thin wrapper that swaps out the prompt-passing mechanism. Everything else — the Stop hook that re-feeds the prompt on each iteration, the `<promise>` detection that ends the loop, the iteration counter — comes from upstream `ralph-loop`. Install upstream first, then this plugin.

## Usage

```text
/ralph-loop-file <path-to-prompt-file> [--max-iterations N] [--completion-promise 'TEXT']
```

| Argument | Required | Description |
|---|---|---|
| `<path-to-prompt-file>` | yes | Path to a file whose contents become the loop prompt. Read via `cat`, so newlines / apostrophes / backticks / dollar signs are all safe. Avoid spaces in the path itself — the slash command still tokenizes the path. |
| `--max-iterations N` | no | Stop after `N` iterations. Default: unlimited. **Always set this** — see "Stopping" below. |
| `--completion-promise 'TEXT'` | no | The exact phrase that ends the loop when the agent emits it inside `<promise>...</promise>` tags. **Always quote the value** if it contains spaces. |
| `--prompt-file PATH` | no | Alternate way to pass the path (the slash command injects this automatically; you usually don't write it yourself). |
| `-h`, `--help` | no | Print full usage. |

### Quick example

Write your task to a file:

```bash
cat > .claude/ralph-task.md <<'EOF'
Fix the bug where `useEffect`'s cleanup function isn't called on unmount.

Steps:
1. Reproduce with the test in `__tests__/cleanup.test.tsx`
2. Make it green without modifying the test
3. Output the completion promise when done
EOF
```

Then in Claude Code:

```text
/ralph-loop-file .claude/ralph-task.md --completion-promise 'BUG FIXED' --max-iterations 15
```

The loop runs in your current session. On each iteration the agent receives the prompt fresh from the file, does work, and either emits `<promise>BUG FIXED</promise>` (loop ends) or doesn't (loop continues until `--max-iterations`).

## The completion promise — how the loop ends

The Stop hook (inherited from upstream `ralph-loop`) inspects the agent's last response on every iteration. It looks for **the exact phrase** you passed to `--completion-promise`, wrapped in `<promise>...</promise>` tags.

If the agent emits:

```text
<promise>BUG FIXED</promise>
```

…the hook exits cleanly and the loop stops.

If the agent emits the phrase **without** the tags, or with a different phrase, or doesn't emit it at all — the loop continues to the next iteration.

This means:

- **The exact text must match.** `BUG FIXED` ≠ `Bug fixed`. Case-sensitive.
- **The `<promise>` tags are required.** The agent saying "I am done — BUG FIXED" without tags doesn't end the loop.
- **The agent must understand the contract.** Your prompt file should tell the agent how to signal completion. The agent doesn't know what `--completion-promise` is set to — you have to spell it out in the prompt.

A prompt that handles this well usually ends with something like:

> When the task is complete, output exactly: `<promise>BUG FIXED</promise>`
> Only emit this when the test is genuinely green. The loop continues if you don't.

## Stopping a running loop

Three ways the loop ends:

1. **The promise** — agent emits `<promise>YOUR_TEXT</promise>` and the Stop hook detects it. Clean exit.
2. **Iteration limit** — `--max-iterations N` reached. Loop stops with a "max iterations" note in the state file.
3. **Manual cancel** — run `/ralph-loop:cancel-ralph` (from the upstream plugin) or `/exit` from inside the session. Work the agent did is already on disk; only the loop driver stops.

**Always pass `--max-iterations`.** A loop without it runs forever in principle — if the agent can't reach the completion condition, the only stop is your manual intervention. 10–20 is a reasonable default for most tasks.

## Monitoring a running loop

The loop's state lives in `.claude/ralph-loop.local.md` (relative to the cwd you started in). Quick checks:

```bash
# Current iteration count
grep '^iteration:' .claude/ralph-loop.local.md

# Full state header
head -10 .claude/ralph-loop.local.md
```

The state file is created on first invocation and updated on every iteration. Add `.claude/ralph-loop.local.md` to your project's `.gitignore` — it's runtime state, not source.

## When to use which command

| Prompt shape | Use |
|---|---|
| Short, single-line, no shell-special chars | `/ralph-loop` (upstream) |
| Multi-line, apostrophes, backticks, code blocks, dollar signs | `/ralph-loop-file` (this plugin) |

When in doubt, write the prompt to a file and use `/ralph-loop-file`. The file-backed path has no failure mode the positional path doesn't also have.

## Known limitation — stop-hook race condition

The upstream Ralph Stop hook (`claude-plugins-official/plugins/ralph-loop/hooks/stop-hook.sh`) occasionally fails to detect a valid `<promise>X</promise>` emission. This affects both `/ralph-loop` and `/ralph-loop-file` — they share the same hook.

**Symptom:** the agent outputs `<promise>BUG FIXED</promise>` correctly, but the loop continues to the next iteration instead of stopping.

**Cause:** the hook reads the JSONL transcript via `grep '"role":"assistant"' | tail -n 100 | jq -rs '... | last'` to extract the most recent assistant text block. The hook fires ~200ms after the agent's promise text is written. If the final content block hasn't fully flushed to disk yet, `jq` reads a snapshot that doesn't include it — `last` returns the previous block (without the promise tag), and the hook concludes "no promise emitted."

**Mitigation:** the race is intermittent. The agent typically re-emits the promise on the next iteration, and the loop eventually converges (or hits `--max-iterations`). Practical advice:

- Set `--max-iterations` to a value that tolerates 1–2 race-induced extra iterations. 15 is plenty for most bugs.
- If the race triggers when the work is already genuinely done, exit manually with `/exit` or `/ralph-loop:cancel-ralph`. The agent's work is already on disk.

A proper fix (patching the hook to scan the last N text blocks instead of just `last`, or adding a small flush delay) requires either forking the upstream plugin or replacing it — both bigger projects than this one repo.

## Files in this repo

```
.claude-plugin/plugin.json     # plugin manifest
commands/ralph-loop-file.md    # the /ralph-loop-file slash command definition
scripts/setup-ralph-loop-file.sh  # parses flags, creates the state file
README.md
LICENSE
```

The setup script's `--help` (`/ralph-loop-file --help` inside Claude Code, or `./scripts/setup-ralph-loop-file.sh --help` on the command line) prints the full flag reference.

## Related

- [`ralph-loop` (upstream)](https://github.com/anthropics/claude-code-plugins/tree/main/ralph-loop) — the original positional-prompt version. This plugin reuses its Stop hook and core loop semantics.
- [`bug-fix-loop` skill](https://github.com/RonenMars/dotfiles/tree/main/claude/skills/bug-fix-loop) — a higher-level Claude Code skill that wraps `/ralph-loop-file` for the specific case of "TDD a Playwright bug to green". Gathers the e2e test dir, project root, and bug description as inputs; substitutes them into a template; writes to `.claude/ralph-task.md`; invokes `/ralph-loop-file` against that file.
- [`ronenmars` marketplace](https://github.com/RonenMars/claude-marketplace) — the marketplace that publishes this plugin alongside `npm-registry-manager` and `npm-package-owner`.

## Contributing

The plugin is small and the design intentionally minimal — most of the surface area is in the upstream `ralph-loop`. Reasonable changes:

- New flags to `setup-ralph-loop-file.sh` that don't conflict with upstream
- Documentation improvements
- Bug fixes in argument parsing

Things to discuss before opening a PR:

- Anything that diverges from upstream `ralph-loop`'s semantics (the value of this plugin is being a thin, predictable variant)
- Adding new state files or `.claude/`-relative paths

To test changes locally without pushing:

```bash
# Clone, then point Claude Code at the local dir as a marketplace
cd ~/path/to/clone
# In Claude Code: /plugin marketplace add ~/path/to/clone
```

(This works because `claude-marketplace` declares this plugin via `"source": "github"` — but a local clone is still a valid plugin directory.)

## License

[MIT](./LICENSE)
