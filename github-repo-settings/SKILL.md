---
name: github-repo-settings
description: >-
  GitHub repo settings via `gh`: labels, merge methods, branch deletion,
  the default branch's ruleset, Pages, a public repo's security features.
  TRIGGER when: `.github/release.yaml` is created or its labels change; user
  asks to sync labels or create/delete a single label, change merge methods,
  toggle head-branch deletion, or enable GitHub Pages; a repo's CI first runs
  on a PR; a job of the `quality` or `security` workflow is added, renamed or
  removed; user asks about branch protection, rulesets or required checks, or
  Scorecard's Branch-Protection check; a repo gains an
  `actions/deploy-pages` workflow; a Pages deploy fails because Pages is off;
  a repository is or becomes public, or gains a `SECURITY.md`; user asks
  about secret scanning, push protection, Dependabot alerts or security
  updates, or private vulnerability reporting.
  SKIP when: workflow YAML with no repo setting involved
  (`github-actions-conventions`).
---

# GitHub repository settings (`gh` CLI)

Administer GitHub repo settings from the CLI. All commands use `gh`, so the repo
is inferred from the current directory's `origin` remote unless `--repo` is given.

## Sync labels to `.github/release.yaml`

`.github/release.yaml` only governs **PR labels** — the changelog generator reads
the labels on merged PRs, nothing else. So it is **not** the full set of labels a
repo should have. Two distinct purposes coexist:

- **Issue-triage labels** (`bug`, `enhancement`, `question`, `help wanted`, …):
  classify *reported* problems/requests, whether or not the repo cuts releases.
  They legitimately exist even when absent from `release.yaml`. **Do not delete
  them just because the changelog config doesn't reference them.**
- **PR / changelog labels** (`feature`, `fix`, `chore`, `dependencies`, …):
  classify the *change shipped*. These category labels are what `release.yaml`
  maps to categories. Area and ecosystem labels (`rust`, …) stay out of it and
  never select a category (see `github-pr-conventions`).
  **Without `release.yaml`, none are created**: nothing reads them (see
  `github-pr-conventions`), so the sync below does not apply. The labels on
  Renovate's PRs are its config's business (see the notes).

The same concept often has both: `bug` (issue) ↔ `fix` (PR), `enhancement` (issue)
↔ `feature` (PR). Inside one changelog category, listing both members of such a
pair is a duplicate — pick one (the PR-side label) and drop the other from
`release.yaml`; but keep the issue-side label alive on the repo. That is also
what `github-pr-conventions` assumes when it labels a PR: `fix`/`feature` by
default, `bug`/`enhancement` only where a repo still maps them.

The `"*"` catch-all in `release.yaml` is a wildcard, **not** a real label — never
create it.

Procedure:

1. Read `.github/release.yaml`, collect every concrete label it maps (drop `"*"`).
   These must all exist on the repo.
2. List current labels: `gh label list --limit 100`.
3. Create any release.yaml label that's missing. Add any repo label missing from
   release.yaml into the right category (or `exclude`) **if it's a category
   label**; leave issue-triage and area/ecosystem labels out of `release.yaml`
   but keep them on the repo.
4. Only delete a label when the user actually wants it gone — never purely because
   it's absent from `release.yaml`.

The block below is a superset: keep `breaking`, the issue-triage labels, and
only the category and exclude labels the repo's `release.yaml` maps (see the
notes after it).

```bash
gh label create feature       --description "New feature"                   --color 0e8a16 --force
gh label create fix           --description "Bug fix"                       --color d73a4a --force
gh label create chore         --description "Maintenance / housekeeping"    --color fef2c0 --force
gh label create dependencies  --description "Dependency updates"            --color 0366d6 --force
gh label create documentation --description "Documentation only"            --color 0075ca --force
gh label create breaking      --description "Backwards-incompatible change" --color b60205 --force
gh label create security      --description "Security fix"                  --color b60205 --force
gh label create deprecation   --description "Deprecates a feature, removed in a later release" --color fbca04 --force
gh label create performance   --description "Performance improvement"       --color 5319e7 --force
# the exclude label, only when release.yaml has an `exclude.labels` entry
gh label create ignore-for-release --description "Exclude from release changelog" --color ededed --force
# issue-triage labels stay even though release.yaml never maps them
gh label create bug         --description "Something isn't working" --color d73a4a --force
gh label create enhancement --description "New feature or request"  --color a2eeef --force
gh label list --limit 100   # verify
```

Notes:

- **`breaking` is always created alongside `release.yaml`.** If the file has
  no *Breaking changes* category, add one — as the **first** category, since
  GitHub files a PR under the first match and a breaking PR also carries its
  `feature`/`fix` label (see `github-actions-conventions` for the baseline file).
- Otherwise, only create the category labels the repo's `release.yaml` actually
  maps; the exclude label only makes sense alongside an `exclude.labels` entry.
- **Create the labels Renovate's config names** (`labels`, `addLabels`;
  `dependency-update-conventions`), reusing an ecosystem label the repo
  already has rather than a variant: two labels for one concept split every
  filter in two.
- `gh label create --force` upserts, so it is safe to re-run.
- Deleting a label removes it from every issue/PR it is on and **cannot be
  undone** — it is destructive. Absence from `release.yaml` is **not** a reason to
  delete; only do it when the user explicitly wants the label gone, and confirm
  the delete list first.
- Leave GitHub's default `bug` / `enhancement` colours and descriptions as-is
  unless asked to unify them with the new labels.

## Restrict merge methods to squash-only

```bash
gh repo edit --enable-squash-merge --enable-merge-commit=false --enable-rebase-merge=false
gh repo view --json mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed   # verify
```

Expected verification output:

```json
{"mergeCommitAllowed":false,"rebaseMergeAllowed":false,"squashMergeAllowed":true}
```

This greys out the "Create a merge commit" and "Rebase and merge" buttons on
PRs, leaving squash as the only option. Adjust the flags to allow other methods
(e.g. `--enable-rebase-merge` to add rebase back).

## Auto-delete head branches on merge

```bash
gh repo edit --delete-branch-on-merge
gh repo view --json deleteBranchOnMerge   # verify -> {"deleteBranchOnMerge":true}
```

Once on, merging a PR deletes its head branch automatically (the branch is still
recoverable from the PR's "Restore branch" button for a while). Pass
`--delete-branch-on-merge=false` to turn it back off.

## Protect the default branch with a ruleset

Every repository with CI protects its default branch with a **ruleset**:
no deletion, no force-push, changes through a pull request, and the
unfiltered CI jobs required to merge. A ruleset, never a classic branch
protection rule: `ossf/scorecard-action`'s `GITHUB_TOKEN` can read a
ruleset but not a classic rule's details, which need an admin token, so
Scorecard's Branch-Protection check cannot grade a classic rule.

Set it **once the CI has run on a first pull request**, so the required
checks carry the names the runs actually report. A required check that
never reports blocks every pull request.

Check first, the answer decides between create and update:

```bash
gh api repos/{owner}/{repo}/rulesets --jq '.[] | {id, name, target}'
```

The ruleset, with the repository's own checks, in a `ruleset.json` outside
the repository:

```json
{
  "name": "default branch",
  "target": "branch",
  "enforcement": "active",
  "conditions": {"ref_name": {"include": ["~DEFAULT_BRANCH"], "exclude": []}},
  "bypass_actors": [{"actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "always"}],
  "rules": [
    {"type": "deletion"},
    {"type": "non_fast_forward"},
    {"type": "pull_request", "parameters": {
      "required_approving_review_count": 0,
      "dismiss_stale_reviews_on_push": true,
      "require_code_owner_review": false,
      "require_last_push_approval": false,
      "required_review_thread_resolution": false
    }},
    {"type": "required_status_checks", "parameters": {
      "strict_required_status_checks_policy": false,
      "required_status_checks": [
        {"context": "lint", "integration_id": 15368},
        {"context": "test", "integration_id": 15368}
      ]
    }}
  ]
}
```

No branch-targeting ruleset yet; create it:

```bash
gh api -X POST repos/{owner}/{repo}/rulesets --input ruleset.json
```

One exists; update it rather than adding a second:

```bash
gh api -X PUT repos/{owner}/{repo}/rulesets/<id> --input ruleset.json
```

Verify: the rules that apply to the default branch, required checks
included.

```bash
gh api repos/{owner}/{repo}/rules/branches/<default-branch>
```

Notes:

- **Required checks: every job of `quality` and, when it exists,
  `security`**, the workflows never filtered by `paths`
  (`github-actions-conventions`). A job skipped on pull requests by an `if:`
  stays out, for the same reason as a filtered one. `context` is the job's
  name as `gh pr checks <number>` prints it, one entry per matrix leg
  (`integration (latest)`); `integration_id` 15368 is GitHub Actions, so a
  status from anything else cannot satisfy it.
- **Renaming, adding or removing a job of `quality` or `security` updates
  the ruleset in the same change**: once that pull request's CI has
  reported the new names, before it merges. A renamed or removed job
  otherwise blocks every pull request, its own included.
- **0 approvals**: a lone maintainer cannot approve their own pull request.
- **The admin bypass is there for `scripts/release.sh`**
  (`release-script-conventions`), whose push lands the release commit
  straight on the default branch, which the `pull_request` rule rejects. A
  repository without that script sets `"bypass_actors": []`. Scorecard's
  Branch-Protection check docks points for the bypass, a price accepted for
  a release flow that stays one command.
- `strict_required_status_checks_policy` stays `false`: requiring the branch
  to be up to date makes every open pull request catch up and re-run its CI
  after each merge.
- A private repository on GitHub Free does not enforce rulesets, which need
  Pro, Team or Enterprise Cloud there. Say so rather than making the
  repository public.

## Enable GitHub Pages in GitHub Actions mode

Any repo with a workflow that runs `actions/deploy-pages` (the `docs` pipeline
of `github-actions-conventions`) needs Pages switched on with **source "GitHub
Actions"** (`build_type: workflow`), never "Deploy from a branch" (`legacy`).
Without it the `deploy` job fails with a 404 from the Pages API, and a
`legacy` source would publish a `gh-pages` branch instead of the uploaded
artifact. Do it as soon as the workflow lands, not after its first red run.

Check first — the answer decides between create and update:

```bash
gh api repos/{owner}/{repo}/pages --jq '{build_type, html_url, cname}'
```

A 404 means Pages is off; create it:

```bash
gh api -X POST repos/{owner}/{repo}/pages -f build_type=workflow
```

Already on with `"build_type": "legacy"`; switch it:

```bash
gh api -X PUT repos/{owner}/{repo}/pages -f build_type=workflow
```

Verify — `build_type` must read `workflow`:

```bash
gh api repos/{owner}/{repo}/pages --jq .build_type
```

Notes:

- `gh api` fills `{owner}` and `{repo}` from the current directory's `origin`.
- `html_url` in the answer is the public URL; a Starlight site's `site`/`base`
  follow it (see `starlight-conventions`). Leave `cname` alone unless the user
  asks for a custom domain — setting one changes every URL of the site.
- Pages on a private repo needs a paid plan; the POST then fails with a 422.
  Say so rather than making the repo public.

## Turn on a public repository's security features

Every **public** repository has these on: secret scanning with push
protection, Dependabot vulnerability alerts, and private vulnerability
reporting, which the reporting link of its `SECURITY.md`
(`security-policy-conventions`) points at. They are free on a public
repository. Turn them on as soon as a repository is public, or when one is
found off.

Check the visibility first. On a private repository secret scanning needs a
paid GitHub Advanced Security licence and private vulnerability reporting
does not exist; say so rather than making the repository public.

```bash
gh api repos/{owner}/{repo} --jq '{visibility, security_and_analysis}'
```

Secret scanning and push protection:

```bash
gh api -X PATCH repos/{owner}/{repo} --input - <<'EOF'
{"security_and_analysis": {"secret_scanning": {"status": "enabled"}, "secret_scanning_push_protection": {"status": "enabled"}}}
EOF
```

Vulnerability alerts, which Renovate's `vulnerabilityAlerts` reads:

```bash
gh api -X PUT repos/{owner}/{repo}/vulnerability-alerts
```

Dependabot security updates **off** — Renovate opens the security PRs, and
both would open one for the same bump (`dependency-update-conventions`):

```bash
gh api -X DELETE repos/{owner}/{repo}/automated-security-fixes
```

Private vulnerability reporting:

```bash
gh api -X PUT repos/{owner}/{repo}/private-vulnerability-reporting
```

Verify: both `status` fields read `enabled`, the alerts call succeeds (it
answers 404 when they are off), security updates read `false` and private
vulnerability reporting `true`.

```bash
gh api repos/{owner}/{repo} --jq .security_and_analysis
```

```bash
gh api --silent repos/{owner}/{repo}/vulnerability-alerts && echo enabled
```

```bash
gh api repos/{owner}/{repo}/automated-security-fixes --jq .enabled
```

```bash
gh api repos/{owner}/{repo}/private-vulnerability-reporting --jq .enabled
```

Notes:

- Every call is idempotent: re-running it where the feature is already on
  changes nothing.
- Turning one of these off is the user's decision, never the fix for a
  blocked push or a noisy alert: a push that push protection blocks holds a
  secret, and the secret is what gets fixed.
