---
name: fix-mr-comments
description: Find unresolved review comments on the current branch's GitLab merge request (via glab) and fix them. Fetches actionable discussion threads from human reviewers and bots (CodeRabbit/bdBot), proposes a fix per thread, applies them on approval, and replies to each thread describing what changed. Use when the user asks to address MR/PR review comments, "fix the review feedback", or resolve reviewer remarks on the open merge request.
tools: Read, Glob, Grep, Bash, Edit, Skill
user-invocable: true
---

# /fix-mr-comments — Find & fix GitLab merge request review comments

You find the unresolved review comments on a GitLab merge request using the `glab` CLI, propose a concrete fix for each, apply the approved fixes following the team's conventions, and reply to each thread with what changed.

This skill **edits code**. It does NOT resolve threads on GitLab — it replies to each thread so the reviewer can verify and resolve. (Replying, not resolving, is the chosen default; do not call `glab mr note resolve` unless the user explicitly asks.)

**Relationship to `/code-review`:** `/code-review` is read-only and reviews *your* changes against the STYLEGUIDE. This skill goes the other direction — it consumes *reviewers'* comments and fixes them. When a comment is vague ("this feels off", "clean this up"), apply the relevant STYLEGUIDE/CLAUDE.md convention exactly as `/code-review` would (thin controllers, `CachedRepository`, audit logging, `ErrorCodeException`, `<P t="..." />`, `interceptErrors`, etc.). Reuse that lens when deciding *how* to fix.

---

## Flow

The flow is **propose → approve → fix → reply**. Do not edit any file before the user approves the proposal list.

### 1. Resolve the merge request

- If `$ARGUMENTS` contains a number, treat it as the MR iid.
- Otherwise resolve the current branch's MR:
  ```bash
  glab mr view --output json
  ```
  Read the `iid` field. If there is no open MR for the branch, say so in one line and stop.

Capture the iid as `<IID>` for the rest of the run.

### 2. Fetch discussions

```bash
glab api "projects/:id/merge_requests/<IID>/discussions" --paginate
```

This returns an array of discussion threads. Each thread has `id` (the discussion id), `resolvable`, and a `notes` array. The first note carries `author.username`, `body`, `system`, `resolved`, and — for diff comments — a `position` object with `new_path` (file) and `new_line` (line).

### 3. Filter to actionable comments

Parse the JSON (python is available; `jq` may not be). Keep a thread **only if**:

- the first note's `system` is `false` (drop "assigned to…", "requested review…", etc.), **and**
- `resolvable` is `true` (drop the big CodeRabbit summary / walkthrough notes, which are non-resolvable), **and**
- the first note's `resolved` is not `true` (skip already-resolved threads).

Include both human reviewers **and** bot reviewers (`bdBot` / CodeRabbit) — they post the actionable line-level remarks.

Reference parsing snippet:

```bash
glab api "projects/:id/merge_requests/<IID>/discussions" --paginate | python -c "
import json,sys
data=json.load(sys.stdin)
for d in data:
    notes=d.get('notes') or []
    if not notes: continue
    n=notes[0]
    if n.get('system') or not d.get('resolvable') or n.get('resolved'): continue
    pos=n.get('position') or {}
    print('===')
    print('discussion:', d['id'])
    print('author:', (n.get('author') or {}).get('username'))
    print('file:', pos.get('new_path'), 'line:', pos.get('new_line'))
    print('body:')
    print(n.get('body'))
"
```

If nothing is actionable, say "No unresolved review comments on MR !<IID>." and stop.

### 4. Understand each comment

For each actionable thread:

- Open the referenced `file:line` with Read and read enough surrounding context to understand the request.
- CodeRabbit comments often contain a **`Suggested fix`** diff and a **`Committable suggestion`** block — use the diff to understand intent, but **re-derive the fix against the real current code** rather than blindly pasting (indentation/line drift, and the suggestion may predate later commits). Verify the suggestion still applies.
- For a human comment phrased as a question or a vague nudge, decide the fix using STYLEGUIDE/CLAUDE.md conventions (lean on the `/code-review` checklist). If a comment is genuinely ambiguous or you disagree, say so in the proposal and ask — don't guess at a behavioural change.

### 5. Propose (do not edit yet)

Present a numbered list, one entry per thread:

```
N. path/file.ts:LINE — @author
   Comment: <one-line summary of what the reviewer asked>
   Fix: <what you will change, with a short before/after if it clarifies>
```

End with: "Reply with which to apply (e.g. `all`, `1,3`, or `skip 2`)." Then **wait**.

### 6. Apply approved fixes

- Edit each approved file with Edit.
- After editing, run the relevant workspace checks for the touched workspaces (see CLAUDE.md): `yarn workspace <name> tsc` (typecheck) and `yarn workspace <name> lint`. A shared change can break both frontend and backend — typecheck every workspace your change touches. Fix any errors you introduced before moving on.
- **Do not commit or push** unless the user asks.

### 7. Reply to each fixed thread

For every thread you fixed, post a short reply describing the change:

```bash
glab mr note create <IID> --reply <DISCUSSION_ID> -m "<what changed>"
```

- `--reply` accepts the full discussion id or an 8+ char prefix.
- Keep the message specific and short, e.g. `Fixed: isConnectionConfigured() now also checks SANDAY_OAUTH_TENANT and SANDAY_OAUTH_SCOPE.`
- Do **not** resolve the thread (`glab mr note resolve`) unless the user explicitly asks — leave resolution to the reviewer.
- For threads the user chose to skip, do not reply.

### 8. Summarize

Close with a one-line tally: `Fixed N of M threads, replied to N — skipped: <list>. Typecheck/lint: <status>.` Note anything left for the user (commit/push, ambiguous comments awaiting a decision).

---

## Hard rules

- **Propose before editing.** No file changes until the user approves the proposal list.
- **Re-derive, don't paste.** Treat a bot's suggested diff as intent, not as a literal patch — confirm it against the live file.
- **Conventions win.** Fix according to STYLEGUIDE.md / CLAUDE.md; reuse the `/code-review` lens for the *how*. When unsure how a fix should look, you may invoke `/code-review` on the touched files.
- **Typecheck after editing** every workspace you touched (`yarn workspace <name> tsc`) — lint does not typecheck.
- **Reply, never resolve** (unless explicitly told). Skipped threads get no reply.
- **No commit/push/migrations/builds** unless explicitly requested.
- **Bots count.** `bdBot` / CodeRabbit line comments are actionable; only `system` and non-resolvable summary notes are dropped.
