# Branch Protection & Repo Settings

Manual repo configuration required for the CI/CD pipeline to work as designed.

---

## 1. Repo settings (Settings → General)

- **Pull Requests → Allow auto-merge**: ✅ enabled
- **Pull Requests → Allow squash merging**: ✅ enabled (default merge strategy)
- **Pull Requests → Allow merge commits**: ❌ disabled (enforces linear history)
- **Pull Requests → Allow rebase merging**: ❌ disabled (optional; squash-only keeps log clean)
- **Pull Requests → Automatically delete head branches**: ✅ enabled

---

## 2. Branch protection (Settings → Branches)

Apply to `main`, `dev`, `stage`, and `prod`.

### Common to all four branches

- Require a pull request before merging ✅
- Require status checks to pass:
  - `lint-check` ✅
  - `security-scan` ✅
- Require branches to be up to date before merging ✅
- Require linear history ✅
- Do not allow bypassing the above settings ✅
- Restrict who can push to matching branches: only the GitHub App (`config-bot` or equivalent) and admins

### Additional — `prod` only

- Require approvals: **1** (or more) ✅
- Require review from Code Owners ✅
- Dismiss stale pull request approvals when new commits are pushed ✅

---

## 3. GitHub App permissions

The App used for promotion PRs (`CONFIG_BOT_PRIVATE_KEY` / `APP_ID`) needs:

| Permission         | Level | Why                                          |
| ------------------ | ----- | -------------------------------------------- |
| Contents           | Write | Push branches (for image-tag PRs)            |
| Pull requests      | Write | Open / auto-merge promotion PRs              |
| Metadata           | Read  | Default                                      |
| Actions            | Read  | Optional — read workflow runs for status     |

Install the App on this repo.

---

## 4. Required repo secrets

| Name                       | Where              | Used by                              |
| -------------------------- | ------------------ | ------------------------------------ |
| `APP_ID`                   | Repo or org        | Mint App tokens in CD workflows      |
| `CONFIG_BOT_PRIVATE_KEY`   | Repo or org        | Mint App tokens in CD workflows      |
| `SLACK_BOT_TOKEN`          | Repo or org        | `notify-slack` composite action      |

## 5. Required repo variables

| Name                       | Example          | Used by              |
| -------------------------- | ---------------- | -------------------- |
| `SLACK_CHANNEL_CI`         | `C0123456789`    | `10-ci-check.yml`    |
| `SLACK_CHANNEL_CD`         | `C0987654321`    | `2x-cd-*.yml`        |

---

## 6. Auto-merge gotchas

- Auto-merge **silently does nothing** if "Allow auto-merge" is off — confirm step 1.
- Auto-merge waits for **all required status checks** declared in branch protection. If a check name is misspelled in branch protection rules, the PR will sit open forever.
- PRs opened by the built-in `GITHUB_TOKEN` **do not trigger downstream workflows** (e.g., `10-ci-check.yml` won't run on the promotion PR). This is why the CD workflows mint a GitHub App token instead.
- The `stage → prod` PR is **opened with auto-merge disabled** by design — human review is required (CODEOWNERS + branch protection).
