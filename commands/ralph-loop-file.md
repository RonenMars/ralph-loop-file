---
description: "Start Ralph Loop with prompt read from a file (safe for multi-line / quotes / backticks)"
argument-hint: "<path-to-prompt-file> [--max-iterations N] [--completion-promise TEXT]"
allowed-tools: ["Bash(${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop-file.sh:*)"]
hide-from-slash-command-tool: "true"
---

# Ralph Loop (File-Backed Prompt)

Use this variant when your prompt contains newlines, apostrophes, backticks, or any
character that would break shell tokenization. The prompt is loaded via `cat` inside
the setup script, so the shell never tries to parse it.

Workflow:

1. Write your full task instructions to a file, e.g. `.claude/ralph-task.md`.
2. Invoke `/ralph-loop-file .claude/ralph-task.md --completion-promise 'DONE' --max-iterations 10`.

The first positional argument MUST be the path. Remaining args are passed through
to `setup-ralph-loop-file.sh` as flags. Avoid spaces in the path — the slash command
still tokenizes the path itself.

```!
"${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop-file.sh" --prompt-file $ARGUMENTS
```

## How to respond after the shell block runs

**If the user passed `--help` or `-h`** (their `$ARGUMENTS` contains either flag): the shell block already printed the help text. Display the captured shell output to the user verbatim inside a markdown code block, then stop. Do NOT paraphrase, summarize, or add usage hints — the help text is the answer. Do NOT enter any loop behavior.

**Otherwise** (a Ralph loop was set up): Please work on the task. When you try to exit, the Ralph loop will feed the SAME PROMPT back to you for the next iteration. You'll see your previous work in files and git history, allowing you to iterate and improve.

CRITICAL RULE: If a completion promise is set, you may ONLY output it when the statement is completely and unequivocally TRUE. Do not output false promises to escape the loop, even if you think you're stuck or should exit for other reasons. The loop is designed to continue until genuine completion.
