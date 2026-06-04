# Restructure Commits Skill

Reorganize the current branch's commits against `development` so each commit is a self-contained, cherry-pickable unit, and unrelated work is split into its own commit (candidate for a separate MR). A file change must travel together with the caller updates it requires, so every commit compiles and tests on its own.

## Safety rules — read before doing anything destructive

- ALWAYS create the backup branch (Step 6.1) before any `git reset` or rewrite.
- NEVER force-push. The skill stops at local history; the user pushes when ready.
- NEVER skip the per-commit lint check in Step 6.3.
- After the rewrite, the net diff `<backup>..HEAD` MUST be empty. If it isn't, STOP and report — something was dropped.
- If the user does not explicitly approve the plan in Step 5, do not proceed to Step 6.

## Step 1: Scope the branch

1. `git fetch origin development`
2. `git merge-base HEAD origin/development` → call this `BASE`
3. `git log --oneline $BASE..HEAD` to list commits (note any merges from development)
4. `git diff --stat $BASE..HEAD` to list changed files
5. `git rev-parse --abbrev-ref HEAD` to capture the current branch name

## Step 2: Map the change graph

For each modified non-test file under `backend/src/`, `frontend/src/`, or `shared/src/`:

1. Read the file's exports (classes, functions, types).
2. Grep across the modified file set to find which other modified files import or reference those exports.
3. Cluster files that import each other into the same cherry-pickable unit.

Attach test files (`*.spec.ts(x)`, `*.integration-spec.ts`) to the source file they test. Attach migrations to the schema change they support.

## Step 3: Classify each cluster

- **Core refactor** — the main story of the branch. File change + ALL its caller updates go in ONE commit so the build stays green at every point.
- **Unrelated work** — touches a different feature area or different ticket. Becomes its own commit; flag as a candidate for a separate MR.
- **MR-feedback / fix-up** — revises an earlier commit on this branch (e.g. `MR comments`, `Clean up`, follow-up assertion). Squash into the commit it fixes.
- **Test / polish** — pure test additions or formatting; attach to the commit they validate, or stand alone at the end.

## Step 4: Propose a reorganization

Output a numbered plan, e.g.:

```
Commit 1 — IT-3998: introduce personalizedEmailMessage factory
  backend/src/communication/personalizedEmailMessage.factory.ts (new)
  backend/src/communication/...

Commit 2 — IT-3998: migrate call sites to personalizedEmailMessage factory
  backend/src/appointments/...   (caller update)
  backend/src/auth/...           (caller update)
  + matching *.spec.ts updates

Commit 3 — IT-XXXX: <unrelated change>          [candidate: split into separate MR]
  backend/src/<other-area>/...
```

Every proposed commit must:
- Carry an `IT-XXXX:` prefix matching this repo's convention.
- Be self-contained (compiles + tests pass on its own).
- Group file + caller updates so cherry-picking never produces a broken intermediate state.

## Step 5: Confirm with the user

Print the plan and ask the user to approve before any history rewrite. If they decline or request edits, stop and revise — never proceed silently. Do not move to Step 6 without explicit approval.

## Step 6: Execute (only after explicit approval)

1. **Backup**: `git branch <current-branch>-backup-<YYYYMMDD-HHMM>` so the original history is recoverable.
2. **Flatten**: `git reset --soft $BASE` — keeps all changes staged, discards old commit boundaries (including any development-merges on the branch).
3. **For each proposed commit, in order**:
   - `git reset` (unstage all)
   - `git add <explicit file paths for this commit>` (never `-A` or `.`)
   - `git commit -m "IT-XXXX: …"`
   - Run the relevant lint check on the files in this commit:
     - backend files → `yarn workspace mijnpraktijk-backend lint`
     - frontend files → `yarn workspace mijnpraktijk-frontend lint`
   - If lint fails, STOP and surface the issue. Do not push through.
4. **Final verification**:
   - `git log --oneline $BASE..HEAD` — confirm the new structure.
   - `git diff <backup>..HEAD --stat` — MUST be empty. Non-empty means content was dropped; restore from the backup branch and report.
5. Leave the push to the user.
