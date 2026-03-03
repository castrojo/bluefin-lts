# Promotion Workflow Errors - Investigation Findings

**Date**: 2026-03-03  
**Status**: Investigated, fixes needed  
**Investigator**: claude-sonnet-4.6 via OpenCode  
**Worktree**: `~/.config/superpowers/worktrees/bluefin-lts/fix-branch-pollution`

---

## Context

Two plans were implemented to fix the LTS promotion and tag publishing pipeline:

1. `2026-02-14-weekly-lts-promotion.md` — Automated weekly promotion workflow (`promote-lts.yml`)
2. `2026-03-02-fix-lts-tag-publishing.md` — Fix accidental production tag publishes from pull bot PRs
3. `2026-03-02-fix-branch-pollution.md` — Replace pull app with manual `promote-to-lts.yml` workflow

Despite these plans being implemented (commits `8ed6d20`, `0b6baa9`, `b144e1a`), builds on the `lts` branch are still **failing**.

---

## Errors Found

### Error 1: `Push Manifest` fails with "image not known" on lts push builds

**Symptom:**  
Runs triggered by `push` to `lts` (e.g. run `22602290633`) fail at the `manifest` job with:
```
Error: retrieving local image from image name ghcr.io/ublue-os/bluefin:lts: ghcr.io/ublue-os/bluefin:lts: image not known
Process completed with exit code 125
```

**Root Cause:**  
In `reusable-build-image.yml`, the `Push Manifest` step (line 495) uses:
```yaml
if: github.event_name != 'pull_request'
```

This is **independent of the `publish` input**. When a push lands on `lts`, `publish=false` (correctly), so `build_push` skips the `Push to GHCR` step and does NOT upload digest artifacts. But the `manifest` job runs anyway (it has `if: always()`), and `Push Manifest` fires because `event_name != 'pull_request'`.

The `Push Manifest` step tries to push `ghcr.io/ublue-os/bluefin:lts` but there's nothing to push — the image was never loaded locally because publish was false — causing the "image not known" error.

**The `sign` job** has the same bug: `if: github.event_name != 'pull_request'` (line 515), independent of `inputs.publish`.

**Fix Required:**  
Change both conditions to respect `inputs.publish`:
```yaml
# Push Manifest step (line 495):
if: inputs.publish

# sign job (line 515):
if: inputs.publish
```

---

### Error 2: `schedule:` trigger still present in all 5 build workflows

**Symptom:**  
All 5 build workflows (`build-regular.yml`, `build-dx.yml`, `build-gdx.yml`, `build-regular-hwe.yml`, `build-dx-hwe.yml`) still have:
```yaml
schedule:
  - cron: '0 2 * * 0'  # Weekly on Sunday at 2 AM UTC
```

**Root Cause:**  
The plan `2026-03-02-fix-lts-tag-publishing.md` specified removing `schedule:` from all build workflows and consolidating it into `scheduled-lts-release.yml` (the dispatcher). However the schedule entries were **never removed** from the individual build workflows.

**Impact:**  
On Sunday at 2 AM UTC, GitHub fires the cron on the **default branch (`main`)**, not `lts`. The publish condition `(github.event_name == 'workflow_dispatch' && (github.ref == 'refs/heads/lts' || github.ref == 'refs/heads/main'))` does NOT match `schedule` events. So the schedule trigger runs builds that do not publish — but it still wastes runner minutes on 5 × 2 = 10 unnecessary builds.

More critically, the dispatcher `scheduled-lts-release.yml` **also runs**, triggering the builds again via `workflow_dispatch` on `lts`. So each Sunday, 20 builds run instead of 10.

**Fix Required:**  
Remove the `schedule:` block from all 5 build workflows:
```yaml
# DELETE these lines from all build-*.yml:
  schedule:
    - cron: '0 2 * * 0'  # Weekly on Sunday at 2 AM UTC
```

---

### Error 3: `promote-to-lts.yml` deviates from plan design

**Symptom:**  
The `promote-to-lts.yml` workflow (from plan `2026-03-02-fix-branch-pollution.md`) was implemented as a checkout+merge approach instead of the plan's simpler `gh pr create --head main` approach.

**Current (implemented):**
```yaml
- Checkout lts branch
- Create branch promote-main-to-lts-TIMESTAMP
- git merge origin/main
- git push branch
- gh pr create --head promote-main-to-lts-TIMESTAMP
```

**Planned design:**
```yaml
- gh pr create --base lts --head main --title "..."
```

**Root Cause:**  
The implemented version creates an intermediate merge branch. This defeats the purpose of replacing the pull app, which also created merge commits polluting history. The plan was to create a direct `main → lts` PR without any intermediate branch.

**Impact:**  
The intermediate branch approach re-introduces the same commit pollution problem the plan was designed to fix (merge commits from intermediate branch land in `lts` history).

**Fix Required:**  
Rewrite `promote-to-lts.yml` to match the plan's original design using `gh pr create --base lts --head main`. Remove checkout/git steps entirely, only needs `GH_TOKEN` and the `gh` CLI.

---

### Error 4: `manifest` job tag suffix logic broken (branch `fix/manifest-testing-tag-mismatch`)

**Symptom:**  
On branch `fix/manifest-testing-tag-mismatch` (commit `b144e1a`), the manifest job's tag suffix logic was changed to:
```bash
if [ "${REF_NAME}" != "${PRODUCTION_BRANCH}" ] && [ "$EVENT_NAME" == "pull_request" ] || [ "${EVENT_NAME}" == "merge_group" ] ; then
```

**Root Cause:**  
This condition has operator precedence issues: `&&` binds tighter than `||`, so it evaluates as:
```
(REF_NAME != PRODUCTION_BRANCH && EVENT_NAME == pull_request) || (EVENT_NAME == merge_group)
```

This means merge_group events **always** get the testing suffix regardless of branch, and pushes to `main` (the testing branch) **never** get the testing suffix.

**Impact:**  
Testing images pushed from `main` would be tagged `:lts` instead of `:lts-testing`, overwriting production tags.

**Fix Required:**  
The original logic `if [ "${REF_NAME}" != "${PRODUCTION_BRANCH}" ]` was correct. The intent of this branch seems to be preventing testing-suffix on lts push events. The correct fix is:
```bash
if [ "${REF_NAME}" != "${PRODUCTION_BRANCH}" ] || [ "${EVENT_NAME}" == "pull_request" ] || [ "${EVENT_NAME}" == "merge_group" ]; then
```
Or more explicitly, only skip the testing suffix when it's a `workflow_dispatch` on `lts`:
```bash
if [ "${REF_NAME}" != "${PRODUCTION_BRANCH}" ]; then
  if [ "${EVENT_NAME}" != "workflow_dispatch" ]; then
    export TAG_SUFFIX="testing"
    export DEFAULT_TAG="${DEFAULT_TAG}-${TAG_SUFFIX}"
  fi
fi
```

---

## Summary Table

| # | Error | File | Status | Severity |
|---|-------|------|--------|----------|
| 1 | `Push Manifest` ignores `publish=false`, fails on lts pushes | `reusable-build-image.yml:495,515` | **Active - breaks lts builds** | High |
| 2 | `schedule:` still in all 5 build workflows (should be in dispatcher only) | `build-*.yml` | Active - wasted runs | Medium |
| 3 | `promote-to-lts.yml` uses checkout+merge instead of `--head main` | `promote-to-lts.yml` | Active - re-introduces pollution | High |
| 4 | Broken tag suffix condition in `fix/manifest-testing-tag-mismatch` branch | `reusable-build-image.yml:372` | On open branch, not merged | High |

---

## Current State of Repository

- **`main`**: Has all fixes from plans (workflow trigger cleanup, dispatcher, renovate restriction, promote-to-lts)
- **`lts`**: Tracking behind main (promotion PR merged as `7d95440`)
- **`fix/manifest-testing-tag-mismatch`**: Open branch with Error 4 (NOT yet a PR upstream)
- **Active worktree**: `~/.config/superpowers/worktrees/bluefin-lts/fix-branch-pollution` tracks `lts`

---

## Next Steps for Future Agent

1. **Fix Error 1** (highest priority — blocking): In `reusable-build-image.yml`, change `Push Manifest` step condition from `if: github.event_name != 'pull_request'` to `if: inputs.publish`. Same for the `sign` job.

2. **Fix Error 2**: Remove `schedule:` block from all 5 `build-*.yml` files.

3. **Fix Error 3**: Rewrite `promote-to-lts.yml` to use `gh pr create --base lts --head main` directly (no checkout/merge/push). Requires only `pull-requests: write` permission and no `contents: write`.

4. **Review Error 4**: The `fix/manifest-testing-tag-mismatch` branch has broken conditional logic. Either fix the operator precedence or revert the change and reframe the goal.

5. After fixing, run `just check && just lint` and submit PRs to `main` (not `lts` directly).

---

## Reference

- Plan for tag publishing fix: `docs/plans/2026-03-02-fix-lts-tag-publishing.md`
- Plan for branch pollution fix: `docs/plans/2026-03-02-fix-branch-pollution.md`  
- Plan for weekly promotion: `docs/plans/2026-02-14-weekly-lts-promotion.md`
- Key commits: `8ed6d20` (tag fix), `0b6baa9` (branch pollution), `b144e1a` (manifest tag mismatch)
- Failed run example: `22602290633` (lts push, Error 1)
- Upstream repo: `https://github.com/ublue-os/bluefin-lts`
