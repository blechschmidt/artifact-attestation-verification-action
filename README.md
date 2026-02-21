# Artifact Attestation Verification Action

A reusable GitHub Actions workflow that verifies artifact attestations for GitHub releases.

## Overview

This workflow iterates through releases of a specified GitHub repository and verifies that all
published artifacts have been signed by GitHub using `gh attestation verify`. It confirms that
artifacts originate from workflows within the declared repository.

## Usage

```yaml
jobs:
  verify:
    uses: blechschmidt/artifact-attestation-verification-action/.github/workflows/verify-attestations.yml@v1
    with:
      repository: owner/repo   # optional: defaults to the calling repository
      tag: ">=1.2.0"           # optional: literal tag or semver constraint
      limit: 5                 # optional: max number of releases to check
      open-issue: true         # optional: open an issue on failure
    permissions:
      contents: read
      issues: write            # only required when open-issue: true
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `repository` | ❌ | calling repo | Repository to verify in `owner/repo` format. Defaults to the repository that called this workflow. |
| `tag` | ❌ | `''` | Literal release tag (e.g. `v1.2.3`) **or** a semver constraint expression (e.g. `>=1.2.0`, `<2.0.0`, `^1.2.0`, `~1.2`, `>=1.0.0,<2.0.0`). Verifies all releases when omitted. |
| `limit` | ❌ | `0` | Maximum number of releases to check (`0` = all). Ignored when `tag` is a literal tag. |
| `open-issue` | ❌ | `false` | Open a GitHub issue on the calling repository when any artifact fails verification. Requires `issues: write` permission. |

### Semver constraint syntax

When `tag` starts with `>`, `<`, `=`, `^`, or `~`, it is interpreted as a constraint rather than
a literal tag name. All matching releases are verified.

| Expression | Meaning |
|------------|---------|
| `>=1.2.0` | version 1.2.0 and above |
| `<2.0.0` | any version below 2.0.0 |
| `>=1.2.0,<2.0.0` | version range (comma-separated, all must match) |
| `^1.2.3` | `>=1.2.3,<2.0.0` |
| `~1.2.3` | `>=1.2.3,<1.3.0` |
| `~1.2` | `>=1.2.0,<1.3.0` |
| `~1` | `>=1.0.0,<2.0.0` |

## Outputs

| Output | Description |
|--------|-------------|
| `verified` | Number of artifacts verified successfully |
| `failed` | Number of artifacts that failed verification |

## How It Works

1. Resolves the target repository (defaults to the calling repository if `repository` is omitted).
2. Determines which releases to check:
   - **Literal tag** — verifies exactly that one release.
   - **Semver constraint** — fetches all releases via the paginated GitHub API, then filters by the constraint.
   - **No tag** — verifies every release (paginated), optionally capped by `limit`.
3. For each release, downloads all uploaded assets with `gh release download`.
4. Runs `gh attestation verify <artifact> --repo <repository>` on every asset.
5. Produces a step summary with a Markdown table showing ✅/❌/⚠️ per artifact.
6. Emits `verified` and `failed` output counts.
7. If any artifact fails and `open-issue: true`, opens a GitHub issue on the calling repository
   listing every failed artifact and linking to the workflow run.
8. Fails the job if any artifact fails verification.

Releases with no downloadable assets are flagged with ⚠️ in the summary but do not count as
failures.

## Permissions

| Permission | When needed |
|------------|-------------|
| `contents: read` | Always — required to download release assets and call the GitHub API |
| `issues: write` | Only when `open-issue: true` |

## Requirements

- Target repository artifacts must have been attested during their build (e.g. via
  [`actions/attest-build-provenance`](https://github.com/actions/attest-build-provenance)).
- The workflow uses the default `github.token` — no additional secrets are required for public
  repositories.
