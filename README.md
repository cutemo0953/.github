# cutemo0953/.github

Shared GitHub Actions reusable workflows for all repos under
`cutemo0953/*`.

## Why this repo exists

2026-04-26 cross-repo CI fires post-mortem (see
`~/Downloads/postmortem-2026-04-26-ci-fires.md`) showed that
copy-pasted workflow files create copy-pasted bugs:
- 5 repos had the same broken `cache-cleanup.yml` and all failed
  simultaneously on the first scheduled cron
- 4 repos all needed the same wrangler 8000111 bypass
- Fix-once-propagate-N-times was as much work as N independent fixes

Reusable workflows physically eliminate this:
- Logic lives once, in this repo
- Consumer repos only specify trigger + inputs
- Bug-fix-once → all consumers automatically pick up (with version pin)
- Drift impossible

## Repo layout

```
.github/workflows/
  cf-pages-deploy.yml      # workflow_call: vite build + Cloudflare Pages deploy
  cache-cleanup.yml        # workflow_call: cache + artifact cleanup via API
  actionlint.yml           # workflow_call: actionlint warn-only CI gate
README.md                  # this file
LICENSE                    # MIT
```

## How consumer repos use these

Each consumer repo keeps a thin wrapper file that specifies the
trigger and any per-repo inputs:

```yaml
# consumer-repo/.github/workflows/deploy.yml
name: Deploy to Cloudflare Pages

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: cutemo0953/.github/.github/workflows/cf-pages-deploy.yml@v1
    with:
      project-name: my-actual-project-name
    secrets: inherit
```

```yaml
# consumer-repo/.github/workflows/cache-cleanup.yml
name: Cleanup Actions Storage

on:
  schedule:
    - cron: '0 3 * * 0'
  workflow_dispatch:

# Caller MUST declare permissions ≥ reusable's permissions.
# Reusable cannot escalate caller's GITHUB_TOKEN scope.
permissions:
  actions: write

jobs:
  cleanup:
    uses: cutemo0953/.github/.github/workflows/cache-cleanup.yml@v1
```

```yaml
# consumer-repo/.github/workflows/actionlint.yml
name: actionlint

on:
  pull_request:
    paths: ['.github/workflows/**']
  push:
    branches: [main]
    paths: ['.github/workflows/**']

jobs:
  lint:
    uses: cutemo0953/.github/.github/workflows/actionlint.yml@v1
```

## Versioning

- Tag `v1`, `v2`, ... for major releases (breaking changes)
- Tag `v1.1`, `v1.2`, ... for minor / patch releases
- Consumers reference `@v1` (auto-picks latest v1.x.y) — same trade-off
  as pinning a GitHub Action to its major

When a breaking change is needed:
1. Bump major (release `v2`) on this repo
2. Migrate consumers one-by-one to `@v2`
3. Verify each green before next migration

### Compatibility rules — what counts as breaking

**Breaking changes (require major bump):**
- Adding a new **required** input to any reusable workflow
- Adding a new **required** secret to any reusable workflow
- Renaming any input or secret
- Tightening an input type (e.g., `string` → enum)
- Changing the default value of an input where consumers have come to
  depend on the old default
- Removing a workflow file entirely

**Non-breaking changes (minor/patch bump only):**
- Adding a new **optional** input with a default
- Adding a new **optional** secret
- Adding a new step inside an existing workflow (as long as it doesn't
  change observable outputs)
- Bumping a pinned dependency version (e.g., `actions/checkout@v5` →
  `@v6`) — IF the new version is API-compatible
- Adding a new workflow file

**Maybe-breaking (judge case-by-case):**
- Changing a default value of an input — usually breaking, sometimes
  consumers depended on it; bump major to be safe
- Bumping a pinned dependency major version — usually breaking unless
  proven backward-compatible

When in doubt, bump major and migrate consumers explicitly. The whole
point of this repo is to make drift impossible; silent breakage at
consumers contradicts that goal.

## Consumer migration log

| Date | Repo | From → To | Verified |
|------|------|-----------|----------|
| 2026-04-26 | IRehab-Patient-PWA | inline → `@v1` | _pending_ |

## Anti-patterns

- ❌ Floating `@main` ref — supply chain risk; one bad commit here
  breaks all consumers immediately
- ❌ Splitting reusable workflow logic across multiple files just
  because consumers had different shapes — better: more inputs
- ❌ Forking a reusable workflow into a consumer repo "because we
  needed a small tweak" — that's how copy-paste drift restarts. Fix
  in this repo, bump version, migrate consumer

## Known leaky abstractions

GitHub's reusable workflow design has constraints we cannot hide:

1. **Caller permissions ≥ called permissions**
   The `permissions:` block in a reusable workflow cannot escalate the
   caller's `GITHUB_TOKEN` scope. If the reusable needs
   `actions: write` (cache-cleanup) or any non-default scope, the
   caller MUST also declare the same permissions, or the run fails
   with `startup_failure` and zero jobs.
   - cf-pages-deploy needs `contents: read` (default) → caller usually OK
   - cache-cleanup needs `actions: write` → caller MUST declare
   - actionlint needs `contents: read` (default) → caller usually OK
   This is documented per workflow in the consumer wrapper examples
   above. Lesson learned 2026-04-26: the first cache-cleanup pilot
   migration failed with startup_failure for exactly this reason.

2. **Cron triggers don't propagate from reusable**
   Cron must be in the caller. Reusable workflows can only declare
   `workflow_call:`. Means each consumer keeps a thin trigger file.
   Acceptable trade-off for centralized logic.

3. **Secrets must be `inherit`-ed or explicitly passed**
   `secrets: inherit` in caller is the simplest pattern. Explicit
   `secrets: { CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }} }`
   works too if you want to be specific.
