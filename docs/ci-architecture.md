# GitHub Actions CI/CD Architecture

This document is the authoritative reference for all CI/CD behavior. Read it completely before touching any workflow file. Agents repeatedly break the CI system by making changes based on assumptions rather than this documented architecture.

## Workflow Files and Their Roles

| File | Role |
|---|---|
| `build-regular.yml` | Caller — builds `bluefin` image |
| `build-dx.yml` | Caller — builds `bluefin-dx` image (developer variant) |
| `build-gdx.yml` | Caller — builds `bluefin-gdx` image (GPU/AI variant) |
| `build-regular-hwe.yml` | Caller — builds `bluefin` with HWE kernel |
| `build-dx-hwe.yml` | Caller — builds `bluefin-dx` with HWE kernel |
| `reusable-build-image.yml` | Reusable workflow — all 5 callers invoke this |
| `scheduled-lts-release.yml` | Dispatcher — owns the weekly Sunday production release |
| `promote-to-lts.yml` | Creates a PR to merge `main` → `lts` (see below) |
| `generate-release.yml` | Creates a GitHub Release when `build-gdx.yml` completes on `lts` |

## Two Branches, Two Tag Namespaces

| Branch | Tags produced | When published |
|---|---|---|
| `main` | `lts-testing`, `lts-hwe-testing`, `lts-testing-YYYYMMDD`, `stream10-testing`, `10-testing`, etc. | Every push/merge to `main` |
| `lts` | `lts`, `lts-hwe`, `lts-YYYYMMDD`, `stream10`, `10`, etc. | Weekly via `scheduled-lts-release.yml` or manual `workflow_dispatch` on `lts` |

**All tags containing `testing` must be published on every push to `main`.** Production tags must only be published from the `lts` branch.

## The `main` → `lts` Promotion Flow

Promotion and production release are **intentionally decoupled**. There are two separate phases:

**Phase 1 — Promotion (manual, no publishing):**
1. A maintainer triggers `promote-to-lts.yml` via `workflow_dispatch`
2. The workflow opens a PR from `main` targeting `lts` directly (no intermediate branch)
3. A maintainer reviews and merges the PR
4. The merge triggers a `push` event on `lts` — all 5 build workflows run as **validation builds** (`publish=false`). No images are published. This is intentional: it confirms that the merged code builds cleanly on `lts` before the next production release.

**Phase 2 — Production release (automated or manual publishing):**
1. `scheduled-lts-release.yml` fires at `0 2 * * 0` (Sunday 2am UTC), OR a maintainer manually triggers it
2. It dispatches all 5 build workflows via `gh workflow run --ref lts`
3. Those are `workflow_dispatch` events on `lts` → `publish=true` → production tags pushed
4. After `build-gdx.yml` completes on `lts`, `generate-release.yml` creates a GitHub Release

**Why `promote-to-lts.yml` exists:** Automated tools (the old Pull app, AI agents) cannot distinguish merge direction — when they see `lts` is behind `main`, they attempt to "sync" and sometimes merge `lts` → `main`, polluting `main` with old production commits. The workflow enforces the correct direction by always targeting `lts` as the base.

**NEVER merge `lts` into `main`.** The flow is always one-way: `main` → `lts`.

## `publish` Input — How It Is Evaluated

All 5 caller workflows pass the same `publish:` expression:

```yaml
publish: ${{
  (github.event_name == 'workflow_dispatch' && (github.ref == 'refs/heads/lts' || github.ref == 'refs/heads/main'))
  ||
  (github.event_name == 'push' && github.ref == 'refs/heads/main')
}}
```

Full truth table:

| Event | Branch | `publish` | Tags published | Notes |
|---|---|---|---|---|
| `push` | `main` | **true** | `-testing` tags | Normal CI after merge |
| `push` | `lts` | **false** | nothing | Intentional — validation only; production ships via dispatch |
| `workflow_dispatch` | `lts` | **true** | production `:lts` tags | Triggered by `scheduled-lts-release.yml` or manually |
| `workflow_dispatch` | `main` | **true** | `-testing` tags | Manual re-run on main |
| `pull_request` | `main` | **false** | nothing | CI check only |
| `merge_group` | `main` | **false** | nothing | CI check only |

**Push to `lts` runs builds but does not publish — this is intentional.** It validates that promoted code compiles cleanly before the next scheduled release. Do not add publish logic to the `push lts` path.

**`publish` defaults to `false`** in `reusable-build-image.yml`. Callers must explicitly opt in. A caller that omits `publish:` will build but not push anything.

## Tag Suffix Logic in `reusable-build-image.yml`

Tag suffixes are computed in two places:

**`build_push` job** (build step):
```bash
if [ "${REF_NAME}" != "${PRODUCTION_BRANCH}" ]; then
  export TAG_SUFFIX="testing"
  export DEFAULT_TAG="${DEFAULT_TAG}-${TAG_SUFFIX}"
fi
echo "DEFAULT_TAG=${DEFAULT_TAG}" >> "${GITHUB_ENV}"
```

**`manifest` job** (`Add suffixes` step):
```bash
if [ "${REF_NAME}" != "${PRODUCTION_BRANCH}" ]; then
  export TAG_SUFFIX="testing"
  export DEFAULT_TAG="${DEFAULT_TAG}-${TAG_SUFFIX}"
  export CENTOS_VERSION_SUFFIX="${CENTOS_VERSION_SUFFIX}-${TAG_SUFFIX}"
fi
echo "DEFAULT_TAG=${DEFAULT_TAG}" >> "${GITHUB_ENV}"
echo "CENTOS_VERSION_SUFFIX=${CENTOS_VERSION_SUFFIX}" >> "${GITHUB_ENV}"
```

**IMPORTANT**: `TAG_SUFFIX` is set with `export` only — it is **never written to `GITHUB_ENV`**. The `Image Metadata` action uses `${{ env.TAG_SUFFIX }}` in its tags expressions, which will always expand to empty string. This is NOT a bug: `CENTOS_VERSION_SUFFIX` already contains the `-testing` suffix, so all tags are generated correctly. Do not "fix" this by adding `TAG_SUFFIX` to `GITHUB_ENV` — it would produce duplicate suffixes like `stream10-testing-testing`.

## SBOM Attestation Rules

SBOMs are generated and attested **only on the `lts` branch** and **only when publishing**. The attestation uses Sigstore/Rekor. Rekor is an external service that has experienced outages (confirmed 2026-02-24: `Post "https://rekor.sigstore.dev/api/v1/log/entries": giving up after 4 attempt(s)`).

All three SBOM steps in `reusable-build-image.yml` must have **both**:
1. `if: ${{ github.ref == 'refs/heads/lts' && inputs.publish }}`
2. `continue-on-error: true`

**A failed SBOM must never block image publishing.** We prefer published images without SBOMs over no images at all. Do not remove `continue-on-error: true` from any SBOM step.

The `sbom:` input has been removed from `reusable-build-image.yml`. SBOM behavior is controlled entirely by the step conditions above — no external toggle is needed or supported.

## CI Build Process Reference

- **Timeout**: 60 minutes configured in `reusable-build-image.yml` (`build_push` job)
- **Platforms**: amd64, arm64 (matrix-driven)
- **Validation**: `just check` runs before every build
- **Build Command**: `sudo just build [IMAGE] [TAG] [DX] [GDX] [HWE] [KERNEL_PIN]`
- **Rechunk**: Runs on all non-PR builds when `publish=true`
- **fail-fast**: false — both platforms attempt independently
- **`publish` default**: `false` — callers must explicitly opt in

## Workflow Condition Quick Reference

When touching any condition in `reusable-build-image.yml`, use this reference:

| Step / Job | Correct condition |
|---|---|
| SBOM steps (Setup Syft, Generate SBOM, Add SBOM Attestation) | `if: ${{ github.ref == 'refs/heads/lts' && inputs.publish }}` + `continue-on-error: true` |
| Rechunk | `if: ${{ inputs.rechunk && inputs.publish }}` |
| Load Image | `if: ${{ inputs.publish }}` |
| Login to GHCR | `if: ${{ inputs.publish }}` |
| Push to GHCR | `if: ${{ inputs.publish }}` |
| Install Cosign | `if: ${{ inputs.publish }}` |
| Sign Image (build_push job) | `if: ${{ inputs.publish }}` |
| Create Job Outputs | `if: ${{ inputs.publish }}` |
| Upload Output Artifacts | `if: ${{ inputs.publish }}` |
| Push Manifest (manifest job) | `if: ${{ inputs.publish }}` |
| sign job (top-level) | `if: ${{ inputs.publish }}` |
| Sign Manifest (inside sign job) | `if: ${{ inputs.publish }}` |

**Signing must only happen when an image is actually published to the registry.** Any condition other than `inputs.publish` on signing or manifest push steps is wrong.

## `schedule:` Triggers — Ownership Rule

**`scheduled-lts-release.yml` is the sole owner of Sunday 2am UTC production builds.**

The 5 build caller workflows (`build-regular.yml`, `build-dx.yml`, `build-gdx.yml`, `build-regular-hwe.yml`, `build-dx-hwe.yml`) must NOT have `schedule:` triggers. Any `schedule:` event on those workflows fires on `main` (the default branch), evaluates `publish=false`, publishes nothing, and wastes runner time.

If you see `schedule:` in any of the 5 build callers, remove it entirely. Do not move or adjust the cron expression — remove it.

## Available Workflows

- `build-regular.yml` — Standard Bluefin LTS (`bluefin` image)
- `build-dx.yml` — Developer Experience (`bluefin-dx` image)
- `build-gdx.yml` — GPU/AI Developer Experience (`bluefin-gdx` image)
- `build-regular-hwe.yml` — HWE kernel variant of `bluefin`
- `build-dx-hwe.yml` — HWE kernel variant of `bluefin-dx`
- `scheduled-lts-release.yml` — Weekly production release dispatcher (sole owner of Sunday builds)
- `promote-to-lts.yml` — Opens a one-way `main` → `lts` promotion PR
- `generate-release.yml` — Creates GitHub Release after successful GDX build on `lts`
