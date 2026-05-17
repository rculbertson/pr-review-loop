# pr-review-loop

A Claude Code skill that automates the AI reviewer response loop for GitHub PRs.

When you open a PR and an AI reviewer (Gemini, Copilot, etc.) leaves comments, this skill polls for those comments, addresses or disputes each one, pushes changes, requests re-review, and repeats — until the PR is approved. Optionally auto-merges when done.

## Usage

```
/pr-review-loop                   # infer PR from current branch
/pr-review-loop 42
/pr-review-loop 42 --auto-merge
/pr-review-loop 42 --reviewer copilot-pull-request-reviewer[bot] --review-command "/copilot review"
```

### Options

| Option | Default | Description |
|--------|---------|-------------|
| `--auto-merge` | false | Merge PR automatically once approved |
| `--reviewer <username>` | `gemini-code-assist[bot]` | GitHub username of the reviewer bot |
| `--review-command <text>` | `/gemini review` | Comment posted to trigger a new review |
| `--poll-interval <seconds>` | `30` | Seconds between polls when no new comments |

## What it does

1. **Infers the PR** from the current branch if no number/URL is given
2. **Polls** for new comments from the reviewer bot
3. **For each comment**: reads the relevant file(s), then either makes the code change (agree) or posts a specific technical rebuttal (disagree)
4. **Batches** all agreed changes into a single commit + push per poll cycle
5. **Requests re-review** by posting the configurable trigger comment
6. **Loops** until approved — then optionally auto-merges

## Prerequisites

- [Claude Code](https://claude.ai/code)
- [`gh` CLI](https://cli.github.com/) authenticated (`gh auth login`)
- `git` with a remote configured

## Installation

```bash
git clone https://github.com/rculbertson/pr-review-loop
mkdir -p ~/.claude/skills/pr-review-loop
cp pr-review-loop/SKILL.md ~/.claude/skills/pr-review-loop/
```

Then restart Claude Code (or open a new session) and `/pr-review-loop` will be available.
