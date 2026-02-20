# Artifact Attestation Verification Action

A reusable GitHub Actions workflow that verifies artifact attestations for GitHub releases.

## Overview

This workflow iterates through releases of a specified GitHub repository and verifies that all published artifacts have been signed by GitHub using `gh attestation verify`. It confirms that artifacts originate from workflows within the declared repository.

## Usage

```yaml
jobs:
  verify:
    uses: blechschmidt/aritfact-attestation-verification-action/.github/workflows/verify-attestations.yml@main
    with:
      repository: owner/repo        # required
      tag: v1.2.3                   # optional: verify a specific release tag
      limit: 5                      # optional: max number of releases to check (0 = all)
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `repository` | ✅ | — | Repository to verify in `owner/repo` format |
| `tag` | ❌ | `''` | Specific release tag to verify. If empty, all releases are checked. |
| `limit` | ❌ | `0` | Maximum number of releases to check (`0` means all, ignored when `tag` is set) |

## Outputs

| Output | Description |
|--------|-------------|
| `verified` | Number of artifacts verified successfully |
| `failed` | Number of artifacts that failed verification |

## How It Works

1. Fetches the list of releases for the specified repository (or uses the provided tag).
2. For each release, downloads all uploaded assets.
3. Runs `gh attestation verify <artifact> --repo <repository>` on every asset.
4. Reports a summary and fails the job if any artifact fails verification.

## Requirements

- The target repository's artifacts must have been attested during their build via GitHub's attestation features (e.g., `actions/attest-build-provenance`).
- The workflow uses the default `github.token` — no additional secrets are required for public repositories.
