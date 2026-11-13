# ncbudd-pipeline

Shared, reusable GitHub Actions workflows and repo templates for the Ncbudd projects. Every third-party action is
pinned to a full commit SHA with a version comment, every workflow starts from `permissions: {}` and grants each job
only what it needs, and checkouts never persist credentials.

## What is here

| Workflow | What it does |
| --- | --- |
| `ci-python.yml` | Install + pytest, optionally sharded (`shards: '[0,1,2,3]'`, files dealt out by size, no extra dependency) |
| `ci-node.yml` | Lockfile install, then the commands you give (typecheck, build, tests) |
| `ci-android.yml` | Gradle tasks (default tests + lint + debug build), optional Node/Capacitor prepare step |
| `security.yml` | Semgrep CE, Trivy filesystem scan, gitleaks, leak denylist, dependency review (PRs) |
| `codeql.yml` | CodeQL for the languages you list |
| `scorecard.yml` | OpenSSF Scorecard (call weekly) |
| `release-image.yml` | Trivy image scan, then (behind the `production` environment) multi-arch build, push to GHCR, cosign keyless signature, SBOM attestation, build provenance; outputs `digest` |

`templates/` holds the issue forms (bug, feature), PR template (Root cause / Fix / Verification / Release note),
`SECURITY.md`, an example `dependabot.yml`, and `labels.yml` (the label set; it only describes the labels, nothing
applies it).

## Adopting it in a repo

1. Copy from `templates/`: `ISSUE_TEMPLATE/`, `pull_request_template.md` to `.github/`; `SECURITY.md` to the root
   (replace `OWNER/REPO`); trim `dependabot.yml` to the ecosystems the repo uses and put it in `.github/`.
2. Add small caller workflows. Use a plain `pull_request` trigger (never `pull_request_target` with PR code), start
   with `permissions: {}` and grant each calling job what the reusable workflow needs. See the `ember-tide` repo's
   `.github/workflows/` for working examples. For instance:

   ```yaml
   jobs:
     security:
       uses: Ncbudd/ncbudd-pipeline/.github/workflows/security.yml@v1
       permissions:
         contents: read
         pull-requests: read
       secrets:
         LEAK_DENYLIST: ${{ secrets.LEAK_DENYLIST }}
   ```

3. Create the labels from `templates/labels.yml` (for example with `gh label create`).
4. Add the repo secret and settings listed below.

This repo must be pushed to GitHub and tagged `v1` before any caller works, because callers reference
`Ncbudd/ncbudd-pipeline/.github/workflows/<name>.yml@v1`. If this repo is private, set Settings -> Actions -> General
-> Access to "Accessible from repositories owned by the user". Once `v1` exists, the callers can be pinned to the
commit SHA instead (`@<sha> # v1`), which is what Scorecard and zizmor prefer; Dependabot's github-actions ecosystem
then keeps them current.

## The leak denylist

`security.yml` greps tracked files for strings that must never appear in a public repo (names, addresses, IPs).
The list is a repository secret named `LEAK_DENYLIST`, one fixed string per line, never committed. Matches are
logged as `file:line` only. Fork pull requests get no secrets, so the step skips there.

## One-tap release approval

The publish job in `release-image.yml` runs in the GitHub Environment `production`. To make it wait for you:

1. In the repo: Settings -> Environments -> New environment -> name it `production`.
2. Tick "Required reviewers" and add yourself. (Optionally restrict deployment to tags `v*`.)
3. With the GitHub mobile app installed, pushing a release tag sends a notification; approve the deployment with
   one tap. Tests and the Trivy scan have already passed by then, and nothing is published before you approve.

## Verifying a published image

```
cosign verify ghcr.io/<owner>/<image>@<digest> \
  --certificate-identity-regexp '^https://github.com/<owner>/<repo>/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
gh attestation verify oci://ghcr.io/<owner>/<image>@<digest> --owner <owner>
```

## Maintaining the pins

Dependabot (github-actions ecosystem, weekly, 7-day cooldown) bumps the SHAs. To pin by hand:
`gh api repos/<owner>/<action>/git/ref/tags/<tag>`, and if the object type is `tag`, resolve it once more with
`git/tags/<sha>` to get the commit. Use releases at least two weeks old. Semgrep in `security.yml` is a pinned pip
version (`semgrep==...`); bump it by hand or with Dependabot's pip ecosystem if you add a requirements file.
