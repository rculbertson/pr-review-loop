---
name: pr-review-loop
description: >
  Automates the AI reviewer response loop for GitHub PRs. Polls the PR for
  comments from an AI reviewer (Gemini, Copilot, etc.), addresses or disputes
  each comment, pushes changes, requests re-review, and repeats until the PR
  is approved. Optionally auto-merges when all comments are resolved.

  TRIGGER this skill when ANY of these match:
  - The prompt starts with "/pr-review-loop"
  - The user asks to "run the PR review loop", "start automated PR review", or "loop on the PR review"

compatibility: Claude Code. Requires gh CLI and git.
allowed-tools: Bash Read Write Edit Glob Grep Task
metadata:
  author: Ryan Culbertson
  version: "1.0"
disable-model-invocation: true
---

# PR Review Loop

Automates responding to AI reviewer comments on a GitHub PR. Polls for new comments, addresses or disputes them, pushes changes, and requests re-review — looping until approved.

## Argument Parsing

The skill is invoked as:
```
/pr-review-loop [PR_NUMBER_OR_URL] [OPTIONS]
```

Parse the invocation arguments:

| Argument | Default | Description |
|----------|---------|-------------|
| First positional arg | (none) | PR number (e.g. `42`) or full GitHub PR URL |
| `--auto-merge` | false | Merge the PR automatically once approved |
| `--reviewer <username>` | `gemini-code-assist` | GitHub username of the reviewer bot |
| `--review-command <text>` | `/gemini review` | Comment body that triggers a new review |
| `--poll-interval <seconds>` | `30` | Seconds to wait between polls when no new comments |

If no PR number or URL is provided, infer the PR from the current branch by running:
```bash
gh pr view --json number,headRefName,baseRefName,url
```
If that command fails (no associated PR), stop and tell the user no PR was found for the current branch.

## Initialization

1. Resolve the PR number and repo. If a full URL was given, extract the number from the path. Store the PR number as `$PR`.

2. Fetch PR metadata:
   ```bash
   gh pr view $PR --json number,headRefName,baseRefName,title,url,reviewDecision
   ```
   Print the PR title and URL so the user can confirm the right PR is targeted.

3. Create a state file and pre-populate it with already-addressed comment IDs:
   ```bash
   STATE_FILE=$(mktemp /tmp/pr-review-loop-${PR}-XXXXXX.txt)
   ```
   Fetch all inline review comments on the PR and check which ones already have a reply from the authenticated user (i.e. us). Pre-populate `$STATE_FILE` with those IDs so they are skipped:
   ```bash
   # Get our own username
   OUR_LOGIN=$(gh api user --jq '.login')

   # Find inline review comment IDs that already have a reply from us
   gh api repos/{owner}/{repo}/pulls/$PR/comments --jq \
     ".[] | select(.user.login==\"$REVIEWER\") | .id" | while read id; do
     # Check if any reply to this comment is from us
     replied=$(gh api repos/{owner}/{repo}/pulls/comments/$id/replies \
       --jq ".[] | select(.user.login==\"$OUR_LOGIN\") | .id" 2>/dev/null | head -1)
     if [ -n "$replied" ]; then
       echo "$id" >> "$STATE_FILE"
     fi
   done
   ```
   This means that on a fresh session, comments already replied to in a previous session are treated as already processed and will not be re-addressed.

4. Print a summary of the configuration:
   ```
   PR: #$PR — $PR_TITLE
   Reviewer: $REVIEWER
   Re-review command: "$REVIEW_COMMAND"
   Auto-merge: $AUTO_MERGE
   Poll interval: ${POLL_INTERVAL}s
   ```

## Poll Loop

**Do not use the Monitor tool.** It runs in a subshell environment where `gh` and other tools may not be on PATH. Instead, Claude drives the polling loop directly using the Bash tool for each individual `gh` call.

Repeat the following steps until a termination condition is met.

### Step 1: Check approval status

Run via the Bash tool:
```bash
gh pr view $PR --json reviewDecision --jq '.reviewDecision'
```
If the result is `APPROVED`, exit the loop (see Termination).

### Step 2: Fetch new inline comments

Print: `[HH:MM:SS] Fetching inline review comments...`

Run via the Bash tool:
```bash
gh api repos/{owner}/{repo}/pulls/$PR/comments \
  --jq '.[] | select(.user.login=="REVIEWER") | [.id, .path, (.line // .original_line // 0), .body] | @tsv'
```

Parse the TSV output line by line. For each row (fields: `id`, `path`, `line`, `body`):
- Skip if `id` is already in `$STATE_FILE`
- Otherwise: append `id` to `$STATE_FILE`, print `New inline comment on $path:$line (id=$id)`, and add to the list of comments to process this cycle

If no new comments found, print: `No new inline comments.`

### Step 3: Fetch new PR-level review comments

Print: `[HH:MM:SS] Fetching PR-level reviews...`

Run via the Bash tool:
```bash
gh pr view $PR --json reviews \
  --jq '.reviews[] | select(.author.login=="REVIEWER") | select(.state=="CHANGES_REQUESTED" or .state=="COMMENTED") | select(.body != "") | [.id, .state, .body] | @tsv'
```

Parse the TSV output. For each row (fields: `id`, `state`, `body`):
- Skip if `id` is already in `$STATE_FILE`
- Otherwise: append `id` to `$STATE_FILE`, print `New review (state=$state, id=$id)`, and add to the list

If no new comments found, print: `No new PR-level reviews.`

### Step 4: No new comments — wait

If both Step 2 and Step 3 found no new comments, print `Sleeping ${POLL_INTERVAL}s...` and run:
```bash
/bin/sleep $POLL_INTERVAL
```
Run this synchronously (do **not** use `run_in_background`). Wait for it to complete before going back to Step 1. Always use `$POLL_INTERVAL` — do not substitute a different duration.

### Step 5: Process each new comment

For each new comment collected in Steps 2–3, in order:

1. **ID is already marked seen** — it was written to `$STATE_FILE` when discovered. No action needed here.

2. **Read the context** — the comment body references specific files and lines. Use the `Read` tool to read the full relevant file(s) before deciding how to respond. Do not act on a comment excerpt without reading the actual code.

3. **Announce the comment and your intent** — before doing anything, print:
   ```
   Comment from $REVIEWER:
   "$COMMENT_BODY"

   Proposed action: [one sentence describing what you will change, or that you will dispute this]
   ```

4. **Decide: agree or disagree** — based on the comment's suggestion and the actual code:

   **If you AGREE** with the suggestion:
   - Make the code change using `Edit` or `Write` tools
   - Reply inline to the comment thread confirming the fix (same `gh api` call as the disagree path above), e.g. "Fixed — [one sentence describing what was changed]."
   - Note the change for the commit message
   - Do NOT commit yet — collect all agreed changes before committing

   **If you DISAGREE** with the suggestion:
   - Reply inline to the specific comment thread:
     - For inline review comments (have a `$COMMENT_ID`):
       ```bash
       gh api repos/{owner}/{repo}/pulls/$PR/comments/$COMMENT_ID/replies \
         --method POST --field body="I disagree with this suggestion because [specific technical reason]. [Explanation of why the current code is correct or the suggestion doesn't apply in this context]."
       ```
     - For PR-level review comments (have a `$REVIEW_ID`):
       ```bash
       gh api repos/{owner}/{repo}/pulls/$PR/reviews/$REVIEW_ID/comments \
         --method POST --field body="..."
       ```
   - Be specific and technical. Reference the code, the invariants, or the design decision that makes the suggestion incorrect or inapplicable.

### Step 6: Commit and push (if changes were made)

If any code changes were made in Step 5:

1. Review all changes: `git diff`
2. Stage all modified files: `git add -A`
3. Commit with a descriptive message:
   ```bash
   git commit -m "Address $REVIEWER review comments

   $(list the specific issues that were addressed)"
   ```
4. Push: `git push`

If no changes were made (all comments disputed), skip to Step 7.

### Step 7: Request re-review

If changes were pushed **or** any comments were disputed this cycle, post the re-review trigger comment so the reviewer can see and respond to the disputes:
```bash
gh pr comment $PR --body "$REVIEW_COMMAND"
```

Then print a status update to the user:
```
# If changes were pushed:
Pushed changes and requested re-review. Waiting for $REVIEWER to respond...

# If only disputes were posted (no code changes):
Posted disputes and requested re-review. Waiting for $REVIEWER to respond to the disagreements...
```

### Step 8: Check termination

Check the termination conditions (see below). If none are met, go back to Step 1.

## Termination Conditions

**Approved**: If `gh pr view $PR --json reviewDecision --jq '.reviewDecision'` returns `APPROVED`, print a success message and exit the loop.

**No new comments after re-review**: If the poll after requesting re-review finds no new comments from `$REVIEWER` for `3` consecutive poll cycles, treat the review as complete.

**Auto-merge**: If `--auto-merge` is set and the PR is approved or marked as complete:
```bash
gh pr merge $PR --squash --delete-branch
```
Use `--squash` by default. If the merge fails with a squash error (e.g., branch protection requires merge commits), retry with `--merge`.

**Manual stop**: The user can Ctrl+C at any time. On exit, print the path to `$STATE_FILE` so the session can be resumed later if desired.

## Error Handling

- **Rate limit**: If `gh` returns HTTP 429 or a rate-limit message, wait 60 seconds before retrying (not the normal poll interval).
- **Push failure**: If `git push` fails, print the error and stop — do not request re-review or loop further. The user needs to resolve the conflict manually.
- **Reviewer not found**: If after 10 poll cycles no comments from `$REVIEWER` have ever appeared, warn the user: "No comments found from '$REVIEWER' after N polls. Verify the reviewer username is correct."
- **gh not authenticated**: If `gh` commands fail with an auth error, stop immediately and tell the user to run `gh auth login`.

## Implementation Notes

- **Batch changes**: When multiple comments touch the same file, process them all before committing. One commit per poll cycle, not one commit per comment.
- **Read before editing**: Always use the `Read` tool to see the current file state before making edits. Comment line numbers may be stale if earlier commits have shifted lines.
- **Commit message quality**: Reference the specific issue(s) addressed, not just "address review comments".
- **Reviewer username is case-sensitive and exact**: Match `.author.login` exactly.
- **Do not re-process seen comments**: The state file is the source of truth. Always check it before processing a comment.
- **Stay on the PR branch**: Verify you're on the correct branch with `git branch --show-current` if in doubt.
