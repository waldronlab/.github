# .github

This is the [special `.github` repository](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/customizing-your-organizations-profile) for the [Waldron Lab](https://github.com/waldronlab) GitHub organization. It contains two types of shared resources:

## Organization Profile

[`profile/README.md`](profile/README.md) is displayed on the Waldron Lab's public GitHub organization page. It introduces the lab, its research focus (cancer genomics, microbiome profiling, and biostatistics), and community resources such as [BugSigDB](https://bugsigdb.org/) and the [Microbiome Virtual International Forum](https://microbiome-vif.org/).

## Reusable Workflows

Reusable GitHub Actions workflows live in [`.github/workflows/`](.github/workflows/) and can be called from any repository in the organization.

### `bioc-pr-cmdcheck-pkgdown.yml`

A reusable workflow for Bioconductor R packages that performs the following steps on both pull requests and branch pushes:

1. **`R CMD check`** — runs [`rcmdcheck`](https://rcmdcheck.r-lib.org/) against the package.
2. **BiocCheck** — runs [`BiocCheck`](https://bioconductor.org/packages/BiocCheck/) to enforce Bioconductor-specific guidelines.
3. **Test coverage** — collects coverage with [`covr`](https://covr.r-lib.org/) on `devel` branch pushes and uploads results to [Codecov](https://about.codecov.io/) (optional, requires `CODECOV_TOKEN` secret).
4. **pkgdown site** — builds and deploys a [`pkgdown`](https://pkgdown.r-lib.org/) site to GitHub Pages when pushing to a `RELEASE_*` branch (optional, enabled by default).
5. **Docker image** — builds and pushes a Docker image to Docker Hub when a `Dockerfile` is present and pushing to the `devel` branch (optional, requires `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets).

#### Usage

Call this workflow from a repository in the organization with:

```yaml
jobs:
  bioc-check:
    uses: waldronlab/.github/.github/workflows/bioc-pr-cmdcheck-pkgdown.yml@devel
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```

#### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `cran` | `string` | `https://p3m.dev/cran/__linux__/noble/latest` | CRAN-like repository URL |
| `enable_pkgdown` | `boolean` | `true` | Build and deploy pkgdown site on `RELEASE_*` branch pushes |
| `enable_docker` | `boolean` | `true` | Build and push Docker image on `devel` branch pushes |
| `dockerfile_path` | `string` | `inst/docker/pkg/Dockerfile` | Path to the Dockerfile to build |

### `claude.yml`

A reusable workflow that runs [Claude Code](https://code.claude.com/docs/en/github-actions) on issues and pull requests. Claude responds when someone with **write access** to the repository:

- mentions `@claude` in an issue, an issue or PR comment, an inline PR review comment, or a PR review;
- opens an issue whose title or body contains `@claude`; or
- adds the label `claude` to an issue. Use this instead of assigning the issue to Claude, which GitHub doesn't allow on private organization repositories.

Claude can answer questions, review code, and implement changes on a new `claude/...` branch for you to turn into a PR.

**Authentication and billing.** Runs are billed to the Waldron Lab Claude API workspace through [workload identity federation](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation). Each run exchanges its GitHub OIDC token for an Anthropic token that expires after about 10 minutes. There is no API key or repository secret to set up. The federation rule accepts only repositories in the `waldronlab` organization: a workflow in any other repository that calls this file is refused.

#### Enable it in a repository

1. **Install the Claude GitHub App** on the repository: <https://github.com/apps/claude>. (Organization owners can choose "All repositories" or add repositories one at a time.)
2. **Add this file** as `.github/workflows/claude.yml` in the repository:

   ```yaml
   name: claude
   on:
     issue_comment:
       types: [created]
     pull_request_review_comment:
       types: [created]
     pull_request_review:
       types: [submitted]
     issues:
       types: [opened, labeled]
   permissions:
     contents: write
     pull-requests: write
     issues: write
     id-token: write
     actions: read
   jobs:
     claude:
       uses: waldronlab/.github/.github/workflows/claude.yml@devel
   ```

   The `permissions` block is required: a reusable workflow can't have more permissions than its caller, and `id-token: write` is what allows the federation exchange.
3. **Test it** by commenting `@claude what does this repository do?` on any issue.

#### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `federation_rule_id` | `string` | waldronlab rule | Claude Console federation rule for the calling repository's owner; only needed outside `waldronlab` (see below) |
| `trigger_phrase` | `string` | `@claude` | Phrase that invokes Claude |
| `label_trigger` | `string` | `claude` | Issue label that invokes Claude |
| `claude_args` | `string` | `""` | Extra Claude Code CLI arguments, e.g. `--max-turns 10` |
| `timeout_minutes` | `number` | `30` | Job timeout; also caps spend per run |

Example with inputs:

```yaml
jobs:
  claude:
    uses: waldronlab/.github/.github/workflows/claude.yml@devel
    with:
      claude_args: "--max-turns 10"
      timeout_minutes: 15
```

#### Repositories outside the waldronlab organization

This workflow is public, so repositories owned by other GitHub accounts can call it too. The default federation rule accepts only `waldronlab` repositories, so each additional account needs:

1. Its own federation rule in the Claude Console (**Settings → Workload identity**), matching subject prefix `repo:ACCOUNT/` (keep the trailing slash) or the claim `repository_owner` = `ACCOUNT`, with audience `https://api.anthropic.com`.
2. The Claude GitHub App installed on that account.
3. The caller workflow above, passing the account's rule:

   ```yaml
   jobs:
     claude:
       uses: waldronlab/.github/.github/workflows/claude.yml@devel
       with:
         federation_rule_id: fdrl_...
   ```

| Account | Federation rule |
|---|---|
| `waldronlab` | `fdrl_01X83Zk3FZmVxU51RcTyhhkP` (default) |
| `lwaldron` | `fdrl_014i6mVkMJG7N84HBzdnr9ch` |

Changes to this workflow affect every repository that calls it, including those outside the organization.

#### Notes

- Comments on other issues and PRs also start this workflow, but the job is skipped unless it was invoked. Skipped runs use no runner minutes; they do appear in the Actions tab.
- Workflows triggered from fork pull requests can't get an OIDC token, a GitHub security restriction, so Claude may not run on inline review comments on fork PRs. Comments in the PR conversation thread are unaffected.
- Don't add an `ANTHROPIC_API_KEY` secret or `anthropic_api_key` input to a repository using this workflow: a static key takes precedence over federation, and the key would then be stored in the repository.

