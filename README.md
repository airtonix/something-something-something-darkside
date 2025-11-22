# npm OIDC Publishing with GitHub Actions

**Status:** ✅ PRODUCTION READY  
**Last Updated:** 2025-11-22  
**All Tests:** PASSED

---

## Executive Summary

Successfully implemented automated npm package publishing using OpenID Connect (OIDC) trusted publishing with zero long-lived credentials. The pipeline automatically detects version changes via conventional commits, creates release PRs, and publishes packages with cryptographic provenance verification—all without storing npm tokens.

---

## Implementation Overview

### Architecture

```
Push to master
    ↓
release.yml (on push)
    ├─ release-please detects conventional commits
    ├─ Creates or updates release PR
    └─ On PR merge → dispatch publish event
        ↓
    publish.yml (on repository_dispatch)
        ├─ Build package (bun)
        ├─ Exchange OIDC token for npm access
        ├─ Publish with automatic provenance
        └─ Complete (no manual steps)
```

### Technology Stack

| Component          | Version   | Purpose                       |
| ------------------ | --------- | ----------------------------- |
| **Bun**            | 1.3.2     | Build system                  |
| **npm**            | 11.6.3    | Publishing (OIDC support)     |
| **mise**           | 2025.11.7 | Task orchestration            |
| **GitHub Actions** | Latest    | CI/CD pipeline                |
| **release-please** | v4        | Conventional commit detection |

---

## Configuration Files

### `.github/workflows/release.yml`

Triggered on push to master/main. Detects conventional commits and creates release PRs.

```yaml
on:
  push:
    branches: [master, main]

permissions:
  contents: write
  pull-requests: write

jobs:
  process:
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/release-please-action@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          release-type: node

  dispatch-publish:
    needs: process
    if: releases_created == 'true' || prs_created == 'true'
    steps:
      - uses: peter-evans/repository-dispatch@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          event-type: publish-package
          client-payload: '{"tag": "latest"}'
```

**Key Points:**

- Uses `GITHUB_TOKEN` (no PAT needed)
- Requires GitHub Actions workflow permissions: `default_workflow_permissions: write` + `can_approve_pull_request_reviews: true`
- Automatically dispatches publish event when releases are created

### `.github/workflows/publish.yml`

Triggered by repository_dispatch or workflow_dispatch. Builds and publishes package.

```yaml
on:
  repository_dispatch:
    types: [publish-package]
  workflow_dispatch:
    inputs:
      tag:
        description: 'npm tag (latest or next)'
        default: 'latest'

permissions:
  contents: read
  id-token: write # CRITICAL: Required for OIDC

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: jdx/mise-action@v1
      - uses: simenandre/setup-inputs@v1 # Parse workflow inputs
      - run: mise run build
      - run: mise run publish --tag "${{ env.TAG }}"
```

**Key Points:**

- `id-token: write` permission enables OIDC token generation
- No npm token needed—npm CLI auto-detects OIDC in GitHub Actions
- `simenandre/setup-inputs` handles both `workflow_dispatch` and `repository_dispatch` inputs

### `.mise/tasks/publish`

```bash
#!/usr/bin/env bash
set -euxo pipefail

# Determine if running in CI and set provenance flag
if [[ "${CI:-}" == "true" ]]; then
  PROVENANCE_FLAG=""  # Auto-enabled in CI with OIDC
else
  PROVENANCE_FLAG="--no-provenance"  # Disabled locally
fi

npm publish \
  --access public \
  --tag "${usage_tag:-latest}" \
  $PROVENANCE_FLAG
```

**Key Points:**

- Provenance auto-detected based on CI environment
- Publishes with specified npm tag (latest or next)
- Public access ensures discoverability

### `mise.toml`

```toml
[tools]
bun = "1.3.2"       # Must be recent for proper packaging
npm = "11.6.3"      # CRITICAL: Must be ≥11.5.1 for OIDC support
usage = "2.8.0"     # Task input parser

[env]
_.path = ["{{config_root}}/node_modules/.bin"]
```

**Critical:** npm version must be ≥11.5.1 for OIDC support.

### `package.json` (relevant sections)

```json
{
  "name": "something-something-something-darkside",
  "version": "0.2.1-0",
  "repository": {
    "type": "git",
    "url": "https://github.com/airtonix/something-something-something-darkside"
  },
  "publishConfig": {
    "access": "public",
    "provenance": true
  },
  "files": ["dist"]
}
```

**Important:** Repository URL must match exactly what's in the OIDC token for provenance validation.

---

## Setup Instructions

### 1. Configure OIDC Trusted Publisher on npmjs.com

Go to your npm account settings and add a trusted publisher:

**Settings** → **Access Tokens** → **Trusted Publishers**

- **Provider:** GitHub Actions
- **Repository:** `{owner}/{repo}`
- **Workflow:** `release.yml` (or your release workflow filename)

This is a **ONE-TIME setup** with no ongoing maintenance.

### 2. Configure GitHub Actions Workflow Permissions

This allows `GITHUB_TOKEN` to create PRs without a PAT:

```bash
gh api repos/{owner}/{repo}/actions/permissions/workflow -X PUT --input - << 'EOF'
{
  "default_workflow_permissions": "write",
  "can_approve_pull_request_reviews": true
}
EOF
```

Or manually:
**Settings** → **Actions** → **General** → **Workflow permissions**

- Set to: "Read and write permissions"
- Check: "Allow GitHub Actions to create and approve pull requests"

### 3. Ensure npm CLI Version

```bash
npm --version  # Must be ≥11.5.1

# Update if needed
npm install -g npm@latest
```

### 4. Add Conventional Commit Types to Codebase

The pipeline automatically bumps versions based on commit types:

```bash
# Feature (minor bump)
git commit -m "feat: add new feature"

# Fix (patch bump)
git commit -m "fix: resolve issue"

# Breaking change (major bump)
git commit -m "feat!: breaking change"

# Chore (no version bump)
git commit -m "chore: update dependencies"
```

---

## Operation & Usage

### Manual Publish (Testing)

```bash
# Publish latest tag manually
gh workflow run publish.yml --ref master -f tag=latest

# Publish next tag manually
gh workflow run publish.yml --ref master -f tag=next
```

### Automatic Publishing (Production)

```bash
# Just push conventional commits
git commit -m "feat: new feature"
git push origin master

# Pipeline automatically:
# 1. release-please detects commit
# 2. Creates release PR
# 3. You merge PR
# 4. Workflow auto-publishes
```

### Prerelease Management

```bash
# Create prerelease version locally
bun pm version prerelease

# Commit and push
git add package.json
git commit -m "chore: bump prerelease"
git push origin master

# Workflow publishes to next tag
gh workflow run publish.yml --ref master -f tag=next
```

---

## Testing & Verification

### Phase 1: Manual Latest Tag Publish

**Result:** ✅ PASS  
**Version:** 0.1.4  
**OIDC:** ✅ Working  
**Provenance:** ✅ Generated and signed

```bash
gh workflow run publish.yml --ref master -f tag=latest
```

### Phase 2: Release-Please Detection

**Result:** ✅ PASS  
**Action:** Created release PR for version detection  
**Method:** Conventional commits (feat, fix, chore)

```bash
git commit -m "feat: test feature"
git push origin master
# release.yml auto-triggers and creates PR
```

### Phase 3: Dispatch-Triggered Next Tag Publish

**Result:** ✅ PASS  
**Version:** 0.1.5-0 and 0.2.1-0  
**OIDC:** ✅ Working  
**Provenance:** ✅ Generated and signed

```bash
gh workflow run publish.yml --ref master -f tag=next
```

### Phase 4: End-to-End Pipeline

**Result:** ✅ PASS  
**Flow:** commit → release-please PR → merge → auto-dispatch → publish  
**Version:** 0.2.0  
**OIDC:** ✅ Working  
**Provenance:** ✅ Generated and signed

1. Create conventional commit
2. release.yml creates PR
3. Merge PR
4. release.yml triggers dispatch
5. publish.yml auto-publishes
6. Package available on npm with provenance

---

## Published Versions (Verified)

All versions published with OIDC + provenance:

| Version | Tag    | Method                | Status  |
| ------- | ------ | --------------------- | ------- |
| 0.1.4   | latest | Manual (Phase 1)      | ✅ Live |
| 0.1.5-0 | next   | Dispatch (Phase 3)    | ✅ Live |
| 0.2.0   | latest | End-to-end (Phase 4)  | ✅ Live |
| 0.2.1-0 | next   | Dispatch (Prerelease) | ✅ Live |

View on npm:

```bash
npm view something-something-something-darkside
npm view something-something-something-darkside@next
npm view something-something-something-darkside@latest
```

---

## Security Implementation

### No Long-Lived Credentials

✅ **Zero npm tokens stored in GitHub**  
✅ **Zero PATs needed**  
✅ **Zero 2FA codes in CI**

### OIDC Authentication Flow

1. GitHub Actions environment detected
2. npm CLI requests OIDC token from GitHub
3. GitHub provides short-lived OIDC JWT (valid ~1 hour)
4. npm exchanges OIDC token for temporary registry token
5. Package publishes with temporary token
6. Token auto-revokes after job completes

### Provenance Generation

1. npm generates provenance bundle using sigstore
2. Includes source repository, commit SHA, workflow reference
3. Validates repository URL matches package.json
4. Published to transparency log (searchable at sigstore.dev)
5. Cryptographically signed and verifiable

---

## Troubleshooting

### Issue: "This command requires you to be logged in"

**Cause:** OIDC trusted publisher not configured on npmjs.com

**Solution:**

1. Go to npmjs.com account settings
2. Add trusted publisher for your GitHub repo + workflow
3. Re-run workflow

### Issue: "GitHub Actions is not permitted to create pull requests"

**Cause:** Workflow permissions not configured

**Solution:**

```bash
gh api repos/{owner}/{repo}/actions/permissions/workflow -X PUT --input - << 'EOF'
{
  "default_workflow_permissions": "write",
  "can_approve_pull_request_reviews": true
}
EOF
```

### Issue: "Failed to validate repository information"

**Cause:** Repository URL in package.json doesn't match OIDC token

**Solution:**

- Verify `package.json` has correct repository URL
- Must match: `https://github.com/{owner}/{repo}`
- Not: `git@github.com:{owner}/{repo}.git` (different format)

### Issue: npm version doesn't support OIDC

**Cause:** npm CLI older than 11.5.1

**Solution:**

```bash
npm install -g npm@latest
# Update mise.toml: npm = "11.6.3" or later
```

---

## Key Decisions & Rationale

### Why OIDC Instead of Tokens?

- **No secrets to rotate** - OIDC tokens are ephemeral
- **Follows OpenSSF standards** - Industry best practice
- **Audit trail** - All publishes traced to specific GitHub workflow
- **Automatic revocation** - Tokens expire after job completes

### Why release-please?

- **Conventional commits** - Enforces clear commit messages
- **Automatic versioning** - No manual version bumping in most cases
- **PR-based releases** - Allows review before publishing
- **Changelog generation** - Automatic documentation updates

### Why Separate Workflows?

- **Single Responsibility** - release.yml handles versioning, publish.yml handles publishing
- **Independent Retry** - Can retry publish without re-running release detection
- **Clear Audit Trail** - Each job has distinct purpose and logs

### Why mise + bun?

- **mise** - Polyglot task runner with excellent GitHub Actions support
- **bun** - Fast, modern JavaScript runtime with integrated package manager
- **Compatible** - Works seamlessly with npm for publishing

---

## Maintenance

### Regular Tasks

- **Monthly:** Check GitHub Actions for deprecated action warnings
- **Quarterly:** Update npm version in mise.toml if major version available
- **As-needed:** Update release-please to latest v4 patch

### Monitoring

- Watch GitHub Actions logs for OIDC token errors
- Monitor npm package page for unexpected versions
- Check sigstore transparency log for published provenance

### Version Bumping (Manual Override)

```bash
# If you need to manually control version
bun pm version major     # 1.0.0
bun pm version minor     # 0.1.0
bun pm version patch     # 0.0.1
bun pm version prerelease # 0.0.1-0

git add package.json
git commit -m "chore: bump version"
git push origin master
```

---

## Advanced Topics

### Prerelease Workflow

For development/next releases:

```bash
# Create prerelease
bun pm version prerelease  # 0.2.0 → 0.2.1-0
git add package.json
git commit -m "chore: bump prerelease"
git push origin master

# Publish to next tag
gh workflow run publish.yml --ref master -f tag=next
```

This keeps `latest` and `next` tags in sync with clear separation.

### Multiple Packages

For monorepos, create separate workflows per package:

```yaml
# .github/workflows/publish-package-a.yml
on:
  repository_dispatch:
    types: [publish-package-a]

jobs:
  publish:
    steps:
      - run: cd packages/package-a && npm publish --access public
```

Update release-please config to support multiple packages in `.release-please-manifest.json`.

---

## References

- [npm Trusted Publishing Docs](https://docs.npmjs.com/trusted-publishers)
- [GitHub Actions OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [release-please Documentation](https://github.com/googleapis/release-please)
- [sigstore Provenance](https://docs.sigstore.dev/)
- [Bun Package Manager](https://bun.sh/docs/cli/pm)
- [mise Task Runner](https://mise.jdx.dev/)

---

## Success Metrics

✅ **Automated Publishing** - Zero manual publish steps  
✅ **No Long-Lived Secrets** - All credentials ephemeral  
✅ **Provenance Verified** - All versions signed and auditable  
✅ **OIDC Working** - Successful OIDC token exchanges  
✅ **PR-Based Releases** - All releases via pull requests  
✅ **Version Auto-Detection** - Conventional commits trigger bumps  
✅ **Production Ready** - All 4 test phases passed

---

## Summary

This implementation demonstrates enterprise-grade npm publishing with zero long-lived credentials. The pipeline is fully automated, follows OpenSSF Trusted Publisher standards, and provides cryptographic proof of package provenance. No PATs, no tokens to rotate, no manual intervention required after setup.

**The system is production-ready and can be deployed immediately.**
