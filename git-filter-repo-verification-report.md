# git-filter-repo verification report

## Summary

This repository was used as a disposable verification target to confirm two `git-filter-repo` use cases:

1. Remove a specific file from Git history.
2. Replace a specific secret string across Git history.

Both scenarios were validated end-to-end: fixture history creation, push to GitHub, history rewrite with `git-filter-repo`, force-push of rewritten history, and verification from a fresh clone.

## Environment

- Repository: `tamagokakedon/git-filter-repo-test`
- Working branch for this report: `tamagokakedon/git-filter-repo-verification`
- Verification branches:
  - `scenario-file-removal-source`
  - `scenario-string-replace-source`
- `git-filter-repo` installation method:
  - `python -m pip install --user git-filter-repo`
- Execution method used in this environment:
  - `python -m git_filter_repo ...`

## What was done

### 1. Install `git-filter-repo`

Command:

```powershell
python -m pip install --user git-filter-repo
```

Result:

```text
Successfully installed git-filter-repo-2.47.0
WARNING: The script git-filter-repo.exe is installed in
'C:\Users\tamag\AppData\Roaming\Python\Python314\Scripts' which is not on PATH.
```

Notes:

- In this environment, `git filter-repo` was not initially available as a Git subcommand.
- The package was installed successfully, but because the script path was not on `PATH`, execution was done with `python -m git_filter_repo`.

### 2. Create verification history

Two isolated branches were created from `main` so the scenarios would not interfere with each other.

#### File removal branch

Commands:

```powershell
git switch -C scenario-file-removal-source main
git add fixtures
git commit -m "Add file-removal verification fixtures"
git add fixtures
git commit -m "Update removable file history"
git push -u origin scenario-file-removal-source
```

Files created:

- `fixtures/file-removal/classified.txt`
- `fixtures/shared/notes.txt`

History created:

```text
083c8fc Update removable file history
b9ded18 Add file-removal verification fixtures
7b5edc9 Initial commit
```

#### String replacement branch

Commands:

```powershell
git switch -C scenario-string-replace-source main
git add fixtures
git commit -m "Add string-replace verification fixtures"
git add fixtures
git commit -m "Expand secret exposure history"
git push -u origin scenario-string-replace-source
```

Files created:

- `fixtures/string-replace/app.env`
- `fixtures/string-replace/notes.txt`
- `fixtures/string-replace/audit.log`

Secret used for verification:

```text
SECRET_TOKEN_ALPHA_12345
```

History created:

```text
7aece49 Expand secret exposure history
69dab31 Add string-replace verification fixtures
7b5edc9 Initial commit
```

## Scenario A: Remove a file from history

### Objective

Remove `fixtures/file-removal/classified.txt` from all history on `scenario-file-removal-source`.

### Rewrite command

```powershell
python -m git_filter_repo --force --invert-paths --path fixtures/file-removal/classified.txt
```

### Pre-rewrite checks

Commands:

```powershell
git --no-pager log --oneline -- fixtures/file-removal/classified.txt
git rev-list --all -- fixtures/file-removal/classified.txt
git grep -n "tracking-id: FILE-REMOVAL-001" $(git rev-list --all)
```

Observed result:

```text
083c8fc Update removable file history
b9ded18 Add file-removal verification fixtures

before_path_commit_count=2
before_tracking_id_hits=2
```

### Rewrite output

Observed result excerpt:

```text
NOTICE: Removing 'origin' remote
HEAD is now at 3a93c29 Update removable file history
```

Important note:

- `git-filter-repo` removed the `origin` remote automatically.
- The remote had to be added again before pushing:

```powershell
git remote add origin https://github.com/tamagokakedon/git-filter-repo-test.git
git push --force origin scenario-file-removal-source
```

### Post-rewrite checks

Commands:

```powershell
git rev-list --all -- fixtures/file-removal/classified.txt
git show HEAD:fixtures/shared/notes.txt
```

Observed result:

```text
after_path_commit_count=0

verification baseline
this file should remain after history rewrite
updated alongside the removable file
```

Fresh clone verification:

```powershell
git clone --branch scenario-file-removal-source --single-branch https://github.com/tamagokakedon/git-filter-repo-test.git <verify-dir>
git rev-list --all -- fixtures/file-removal/classified.txt
git show HEAD:fixtures/shared/notes.txt
```

Observed result:

```text
verify_path_commit_count=0
```

### What this proved

- A specific path can be removed from all reachable history on a branch.
- Commit hashes changed as expected:
  - before: `083c8fc`
  - after: `3a93c29`
- Remaining files in the same repository continued to exist after the rewrite.
- The rewritten history could be force-pushed and observed from a fresh clone.

## Scenario B: Replace a secret string from history

### Objective

Replace the string `SECRET_TOKEN_ALPHA_12345` from all history on `scenario-string-replace-source`.

### Replacement map

Content of the replacement file:

```text
SECRET_TOKEN_ALPHA_12345==>REDACTED_TOKEN
```

### Rewrite command

```powershell
python -m git_filter_repo --force --replace-text replacements.txt
```

### Pre-rewrite checks

Commands:

```powershell
git grep -n "SECRET_TOKEN_ALPHA_12345" $(git rev-list --all)
git grep -n "REDACTED_TOKEN" $(git rev-list --all)
```

Observed result:

```text
before_old_secret_hits=6
before_new_secret_hits=0
```

### Rewrite output

Observed result excerpt:

```text
NOTICE: Removing 'origin' remote
HEAD is now at ae50334 Expand secret exposure history
```

Push-back commands:

```powershell
git remote add origin https://github.com/tamagokakedon/git-filter-repo-test.git
git push --force origin scenario-string-replace-source
```

### Post-rewrite checks

Commands:

```powershell
git grep -n "SECRET_TOKEN_ALPHA_12345" $(git rev-list --all)
git grep -n "REDACTED_TOKEN" $(git rev-list --all)
git show HEAD:fixtures/string-replace/app.env
git show HEAD:fixtures/string-replace/audit.log
```

Observed result:

```text
after_old_secret_hits=0
after_new_secret_hits=6

API_URL=https://example.test
API_TOKEN=REDACTED_TOKEN
FEATURE_FLAG=true
BACKUP_TOKEN=REDACTED_TOKEN

2026-05-27 token seen: REDACTED_TOKEN
2026-05-27 rotation pending
```

Fresh clone verification:

```powershell
git clone --branch scenario-string-replace-source --single-branch https://github.com/tamagokakedon/git-filter-repo-test.git <verify-dir>
git grep -n "SECRET_TOKEN_ALPHA_12345" $(git rev-list --all)
git grep -n "REDACTED_TOKEN" $(git rev-list --all)
```

Observed result:

```text
verify_old_secret_hits=0
verify_new_secret_hits=6
```

### What this proved

- A literal secret string can be replaced across all reachable history.
- Rewritten content was visible in a fresh clone.
- Commit hashes changed as expected:
  - before: `7aece49`
  - after: `ae50334`
- This approach is suitable for dummy secrets and, with proper validation, for real credential cleanup workflows.

## What `git-filter-repo` enabled in this verification

This verification confirmed that the team can use `git-filter-repo` to:

1. Remove accidentally committed files from history.
2. Replace leaked credentials or sensitive strings from history.
3. Rewrite history and republish it to GitHub with force-push.
4. Validate the rewritten result from a clean clone, not only from a local working copy.

It also confirmed these operational characteristics:

- rewritten commits get new hashes
- force-push is required
- local clones and open branches can diverge after the rewrite
- `origin` may be removed by the tool and must be re-added before push

## What should be documented for team operation

The following points should be documented before rolling this out as a team procedure.

### 1. When history rewrite is allowed

Document:

- what kinds of incidents justify a history rewrite
- who can approve it
- whether protected branches need temporary exceptions
- whether the team will rewrite only one branch or all affected refs

### 2. Standard operating procedure

Document a runbook that includes:

1. Identify the affected branch, path, or secret.
2. Freeze merges and pushes temporarily.
3. Create a backup reference before rewriting.
4. Run `git-filter-repo` in a fresh clone.
5. Verify with history-level checks, not only working tree checks.
6. Re-add `origin` if the tool removed it.
7. Force-push the rewritten refs.
8. Notify the team that all old clones and branches may need refresh or reclone.

### 3. Verification checklist

Document exact verification commands such as:

```powershell
git rev-list --all -- <path>
git grep -n "<secret>" $(git rev-list --all)
git --no-pager log --oneline -- <path>
git clone --single-branch --branch <branch> <repo> <verify-dir>
```

The team should explicitly record:

- target path or target secret
- refs that were rewritten
- command used
- result before and after
- final pushed commit hash

### 4. Communication template

Document a message template for the team that explains:

- which branch was rewritten
- why the rewrite happened
- whether credentials were rotated
- whether developers should reclone or hard-reset local branches
- whether PR branches need to be recreated

### 5. Post-incident actions

Document follow-up tasks such as:

- rotate the leaked credential if it was real
- invalidate any tokens before or immediately after rewrite
- update secret scanning or commit hooks
- review `.gitignore`
- review CI logs, artifacts, releases, tags, and external mirrors

### 6. Local recovery instructions for developers

Document one recommended recovery path. In many teams, the safest instruction is:

1. finish or save local work
2. reclone the repository
3. recreate in-progress branches from the rewritten remote history

If the team prefers in-place recovery, document the exact branch reset procedure and when it is safe to use.

## Recommended team-facing documents

At minimum, prepare these documents:

1. **History rewrite runbook**: exact procedure and approvals.
2. **Secret leak response playbook**: rewrite plus credential rotation and notification steps.
3. **Developer recovery guide**: how each developer refreshes local clones after the rewrite.
4. **Verification checklist**: commands and evidence to capture before declaring success.

## Final state

- Current clean branch in this worktree: `tamagokakedon/git-filter-repo-verification`
- Rewritten branch tip for file removal: `3a93c29`
- Rewritten branch tip for string replacement: `ae50334`

This file records the verification results and can be used as the basis for a team runbook.
