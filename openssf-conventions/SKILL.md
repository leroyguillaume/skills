---
name: openssf-conventions
description: >-
  OpenSSF best practices for a public repository: the Best Practices badge,
  `.bestpractices.json`, which skill answers each Scorecard check, and the
  checks left low on purpose.
  TRIGGER when: a repository is or becomes public; creating or editing
  `.bestpractices.json`; user asks about OpenSSF, the OpenSSF Best Practices
  badge (bestpractices.dev, formerly CII), a Scorecard score or finding, or
  how to raise either.
  SKIP when: the repository is private; editing the `scorecard` workflow
  itself (`github-actions-conventions`).
---

# OpenSSF conventions

A public repository is measured against two OpenSSF yardsticks:
[Scorecard](https://github.com/ossf/scorecard/blob/main/docs/checks.md),
which grades it automatically every week, and the
[Best Practices badge](https://www.bestpractices.dev/en/criteria/0), which
its maintainer self-certifies. Both mostly measure practices other skills
already set. This skill routes each check to that skill, and covers the badge
itself.

## Scope

**Public repositories only.** A private repository runs no Scorecard
(`github-actions-conventions`) and applies for no badge: nobody outside can
read either result.

The badge requires a FLOSS licence. A public repository the user has left
unlicensed cannot earn it: say so, never pick the licence for them
(`readme-conventions`).

## Where each Scorecard check is handled

| Check | Home |
| --- | --- |
| Binary-Artifacts, CI-Tests | `ci-conventions` |
| Branch-Protection, SAST, Webhooks | `github-repo-settings` |
| CII-Best-Practices | [below](#the-best-practices-badge) |
| Dangerous-Workflow, Token-Permissions | `github-actions-conventions` |
| Pinned-Dependencies | `github-actions-conventions`, `ci-conventions`, `docker-conventions`, `pre-commit-conventions` |
| Dependency-Update-Tool, Vulnerabilities | `dependency-update-conventions`, `ci-conventions` |
| License | `readme-conventions` |
| Packaging, Signed-Releases, SBOM | `release-supply-chain-conventions` |
| Security-Policy | `security-policy-conventions` |
| Code-Review, Contributors, Maintained, Fuzzing | [left low on purpose](#checks-left-low-on-purpose) |

A finding is fixed in the skill that owns it, and in the repository. A
check missing from this table is new: read its entry in Scorecard's
documentation and give it a home in the skill that owns the subject.

## Checks left low on purpose

Some checks measure the people around a project, not its files. A project
with one maintainer scores low on them, and that is accepted, not worked
around:

- **Code-Review**, and the review tiers of Branch-Protection: a maintainer
  cannot approve their own pull request, hence 0 required approvals
  (`github-repo-settings`). Never add a bot approval or a second account to
  lift the score: Scorecard ignores bot reviews, and a sock puppet defeats
  what review is for.
- **Contributors** and **Maintained** count organisations and recent
  activity. No commit is made to look active.
- **Fuzzing**, only on the user's ask. A fuzz harness pays off on a parser
  of untrusted input, and Scorecard only recognises OSS-Fuzz,
  ClusterFuzzLite, Go's native fuzzing and a few property-testing libraries.

## The Best Practices badge

Every public repository aims at the **passing** level. Silver and gold add
requirements on governance and on the people (two-person review, a bus
factor of two, unassociated contributors): only on the user's ask.

### Registering is the user's step

The badge entry belongs to whoever certifies it: the user signs in at
<https://www.bestpractices.dev> with their GitHub account and adds the
repository. Check first whether it is registered; a `301` names the project
id in its redirect, a `404` means it is not:

```bash
curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' 'https://www.bestpractices.dev/en/projects.json?as=badge&url=https%3A%2F%2Fgithub.com%2F<owner>%2F<repo>'
```

### `.bestpractices.json` proposes the answers

The badge app reads a `.bestpractices.json` at the repository root and
proposes its answers in the form. Write it from the
[passing criteria](https://www.bestpractices.dev/en/criteria/0) as they read
today, never from memory: a `<criterion>_status` (`Met`, `Unmet` or `N/A`)
and a `<criterion>_justification` naming the file that proves it, by URL.

```json
{
  "contribution_status": "Met",
  "contribution_justification": "The contribution process is in https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md",
  "vulnerability_report_process_status": "Met",
  "vulnerability_report_process_justification": "https://github.com/<owner>/<repo>/blob/main/SECURITY.md"
}
```

- **Only what the repository proves.** A criterion about what people know
  or do (`know_secure_design`, `know_common_errors`, `report_responses`,
  `vulnerability_report_response`, `maintained`) stays out of the file: the
  user answers it in the form. An agent never certifies a person.
- **It stays true.** A change that breaks a claim (drops the test suite,
  moves `SECURITY.md`) updates the file in the same change. After a change,
  the user re-runs the proposals with the form's "Save (and continue) 🤖".
- The criteria's non-obvious homes: `release_notes_vulns` is
  `pr-conventions` and `dependency-update-conventions`, `report_process` is
  `contributing-conventions`, `vulnerability_report_response` is
  `security-policy-conventions`, and `static_analysis` is met by the linters
  in `pre-commit` plus CodeQL (`github-repo-settings`).

### The badge in the README

Once the project has an id, its badge joins Scorecard's on the README's
badge line (`readme-conventions`):

```markdown
[![OpenSSF Best Practices](https://www.bestpractices.dev/projects/<id>/badge)](https://www.bestpractices.dev/projects/<id>)
```

Never before the project exists: a badge for an unregistered project is a
broken image.

## Never

- Never apply OpenSSF checks or the badge to a private repository.
- Never game a check that measures people: no bot approval, no second
  account, no commit made to look active.
- Never mark a criterion `Met` in `.bestpractices.json` that the repository
  does not prove, or one about what a person knows or does.
- Never add the badge before the project is registered.
