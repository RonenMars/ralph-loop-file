# ralph-loop-file

A `/ralph-loop-file` slash command for Claude Code — a variant of [`ralph-loop`](https://github.com/anthropics/claude-code-plugins/tree/main/ralph-loop) that reads its prompt from a file instead of taking it as a shell argument.

## Why

The original `/ralph-loop` accepts the prompt positionally on the command line, which means the shell tokenizes prompt content before Claude Code sees it. That breaks the moment your prompt contains:

- Apostrophes (`don't`)
- Backticks (`` `someFunction()` ``)
- Dollar signs (`$variable`)
- Newlines
- Any other shell-special character

`/ralph-loop-file` reads the prompt with `cat` inside its setup script, *after* the shell has finished parsing the slash command's flags. The prompt content never touches shell tokenization.

## Usage

```text
/ralph-loop-file <path-to-prompt-file> [--max-iterations N] [--completion-promise TEXT]
```

The first positional argument must be the path. Remaining args are passed through to `setup-ralph-loop-file.sh` as flags.

### Example

Write your task to a file:

```bash
cat > .claude/ralph-task.md <<'EOF'
Fix the bug where `useEffect`'s cleanup function isn't called on unmount.

Steps:
1. Reproduce with the test in `__tests__/cleanup.test.tsx`
2. Make it green
3. Output "BUG FIXED" when done
EOF
```

Then invoke:

```text
/ralph-loop-file .claude/ralph-task.md --completion-promise "BUG FIXED" --max-iterations 15
```

## Installation

This plugin is published via the [`ronenmars` marketplace](https://github.com/RonenMars/claude-marketplace).

```text
/plugin marketplace add RonenMars/claude-marketplace
/plugin install ralph-loop-file@ronenmars
```

## Completion promise

When you pass `--completion-promise "<TEXT>"`, the loop continues until the agent's response contains that exact text. Use this for "stop only when the task is genuinely done" semantics — not as an early-exit hatch. The agent should output the promise text **only** when the work is fully complete.

## Related

- [ralph-loop (upstream)](https://github.com/anthropics/claude-code-plugins) — the original positional-prompt version. This plugin shares its stop hook and core loop semantics.
- [bug-fix-loop skill](https://github.com/RonenMars/dotfiles/tree/main/claude/skills/bug-fix-loop) — a higher-level skill that wraps `/ralph-loop-file` for the specific case of "TDD a Playwright bug to green".

## License

MIT
