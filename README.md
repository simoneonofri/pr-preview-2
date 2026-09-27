# PR Preview 2

Secure pull-request previews for W3C-style specifications using GitHub Actions, [w3c/spec-prod](https://github.com/w3c/spec-prod), GitHub Pages, and W3C HTML Diff.

**Status:** pre-release / testing.

## What it does

For each pull request, PR Preview 2:

1. builds the exact PR **HEAD** with `w3c/spec-prod`;
2. builds the PR **BASE** at the Git merge-base;
3. publishes immutable snapshots under:
   `/pr-preview/<PR>/<HEAD_SHA>/`;
4. updates:
   `/pr-preview/<PR>/latest/`;
5. generates a W3C HTML Diff between BASE and HEAD;
6. mirrors relative assets into the diff output so images and other local resources resolve correctly;
7. posts or updates a dedicated PR comment;
8. removes the complete `/pr-preview/<PR>/` directory when the PR closes.

The normal GitHub Pages root is not intentionally modified by preview commits.

## Security model

PR Preview 2 is deliberately split into two workflows.

```text
pull_request
    |
    v
Untrusted Build
contents: read
    |
    | artifact
    v
workflow_run
    |
    v
Trusted Publisher
validated artifact
    |
    v
gh-pages + GitHub Pages
```

The build workflow runs PR-controlled source with read-only repository permissions.

The publisher runs from the default branch, never checks out the PR HEAD, re-queries the trusted GitHub API for PR/SHA state, validates the artifact as untrusted data, and only allows preview mutations below `pr-preview/<PR>/`.

## Tested

End-to-end tests have been completed on repositories owned by `simoneonofri` only.

### Bikeshed

Tested on `simoneonofri/threat-modeling-guide`:

- BASE / HEAD builds
- local assets
- W3C HTML Diff
- multiple immutable snapshots
- `latest`
- concurrent PR publishers
- merged cleanup
- closed-unmerged cleanup
- preservation of the existing Pages root

### ReSpec

Tested on `simoneonofri/digital-credentials`.

Live test PR:

https://github.com/simoneonofri/digital-credentials/pull/5

Live preview:

https://simoneonofri.github.io/digital-credentials/pr-preview/5/latest/head/index.html

Live diff:

https://simoneonofri.github.io/digital-credentials/pr-preview/5/latest/diff/index.html

Local SVG resolved from the diff:

https://simoneonofri.github.io/digital-credentials/pr-preview/5/latest/diff/pr-preview-2-test.svg

`static` is **experimental / untested** in PR Preview 2. The underlying `spec-prod` toolchain supports it, but it has not yet been exercised end-to-end here.

## Installation for testing

For the first evaluation, use a fork or non-critical sandbox repository rather than a primary production repository. The trusted publisher intentionally requires write access to `gh-pages` and Pages deployment permissions.

Two small caller workflows must live in the adopting repository.

Copy:

- `examples/pr-preview-build.yml` to `.github/workflows/pr-preview-2-build.yml`
- `examples/pr-preview-publish.yml` to `.github/workflows/pr-preview-2-publish.yml`

Then set the build caller inputs for the repository.

### ReSpec

```yaml
with:
  toolchain: respec
  source: index.html
```

### Bikeshed

```yaml
with:
  toolchain: bikeshed
  source: index.bs
```

## Required integration rules

### 1. The publisher caller must be on the default branch

GitHub only triggers `workflow_run` when the workflow exists on the repository default branch.

### 2. Normal Editor's Draft publication must share the same Pages lock

The job that publishes the normal Editor's Draft must include:

```yaml
concurrency:
  group: pr-preview-pages-${{ github.repository_id }}
  cancel-in-progress: false
```

Without this, a normal Pages deployment and a preview deployment can race.

### 3. GitHub Pages must use GitHub Actions deployment

The repository must have GitHub Pages enabled with **GitHub Actions** as the deployment source.

### 4. A persistent `gh-pages` branch must exist

PR Preview 2 stores the preview archive under `pr-preview/` in that branch and deploys the complete current tree.

## Artifact validation

The trusted publisher rejects payloads that:

- do not match trusted repository / PR / SHA state;
- contain symlinks or special files;
- contain unexpected top-level entries;
- contain more than 5,000 files;
- exceed 50 MiB;
- do not contain both `base/index.html` and `head/index.html`;
- attempt to mutate paths outside `pr-preview/<PR>/`.

Stale builds are skipped if the PR HEAD has already changed.

## Concurrency

PR builds are independent and may run concurrently.

All trusted preview publishers are serialized per repository:

```text
pr-preview-pages-<repository_id>
```

The normal Editor's Draft publisher must use the same group.

## History and cleanup

While a PR is open:

```text
pr-preview/
  123/
    <sha-1>/
      base/
      head/
      diff/
    <sha-2>/
      base/
      head/
      diff/
    latest/
```

When the PR closes, `pr-preview/123/` is removed from the current `gh-pages` tree.

Historical Git blobs remain in Git history. History compaction is intentionally out of scope for the first release.

## Known security constraint

Preview HTML is served from the same GitHub Pages origin as the normal Editor's Draft. A PR can therefore contribute HTML/JavaScript that executes under that Pages origin.

Repositories that store sensitive browser state on the Pages origin should evaluate this constraint before adopting PR Preview 2.

## Dependencies

The reusable workflows pin tested GitHub Actions and `w3c/spec-prod` to immutable commit SHAs.

## Next validation before v0.1

- exercise these reusable workflows from a separate adopting repository;
- test simultaneous normal Editor's Draft + PR Preview publishing;
- run negative artifact validation tests;
- test the `static` toolchain.

