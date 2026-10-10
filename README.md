# Updog_restore release baseline staging

Disposable acceptance repository for Updog_restore ADR-107 (scoped pre-merge
release baseline prerequisite). It holds mock data only. It has no
environments, no secrets, and no production coupling. Never point it at a real
provider.

## What is copied

- `scripts/` and `.github/path-filters.yml`: verbatim from Updog_restore `main`
  at `909671a3f`.
- `.github/workflows/ci-unified.yml`: reduced CI with the change classifier, the
  `Release Baseline Prerequisite` job, and `CI Gate Status`. The classify step,
  the feeder job, and the gate re-fence step are verbatim.
- `.github/workflows/capture-release-baseline.yml`: verbatim, except that a mock
  identity replaces the provider capture step and there is no `Production`
  environment.

Branch protection on `main` mirrors production: the required check is
`CI Gate Status` from GitHub Actions (App 15368), strict updates are on, and
administrators are included. Squash is the only merge method, and auto-merge is
allowed.

## Acceptance scenarios

| ID | Scenario | Expected |
| --- | --- | --- |
| S1 | Open a PR that edits the plan, with no capture. Try a merge through the API and through auto-merge. | The gate is red and both merge paths are blocked. |
| S2 | Dispatch the capture for the exact PR head, then rerun the original `pull_request` CI run. | The feeder passes, the gate turns green, and the merge is allowed. |
| S3 | After S2 is green, push an unrelated commit to `main`. | Strict protection blocks the merge. After the branch update, the head changes and the feeder fails until a new capture. |
| S4 | Run a `workflow_dispatch` CI run on the candidate branch. | The feeder refuses a non-`pull_request` event, so the dispatch cannot satisfy the gate. |
| S5 | Confirm that the required check binds App 15368 and the PR head or test-merge target. | Only the GitHub Actions check on the current head counts. |

Capture dispatch for PR `N`, run by the repository owner:

```bash
gh workflow run capture-release-baseline.yml --ref main \
  -f baseline_main_sha="$(gh api repos/OWNER/REPO/commits/main --jq .sha)" \
  -f planned_pr_head_sha="$(gh pr view N --json headRefOid --jq .headRefOid)" \
  -f pr_number=N \
  -f plan_path=docs/1-plans/F_1.21.1_production-release-reserve-delivery.plan.md \
  -f plan_sha256="$(gh api 'repos/OWNER/REPO/contents/docs/1-plans/F_1.21.1_production-release-reserve-delivery.plan.md?ref=HEAD_SHA' --jq .content | base64 -d | shasum -a 256 | cut -d' ' -f1)"
```

The first `main` push run is red by design: a new branch has no `before` commit
for the classifier.
