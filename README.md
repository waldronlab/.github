# .github

This is the [special `.github` repository](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/customizing-your-organizations-profile) for the [Waldron Lab](https://github.com/waldronlab) GitHub organization. It contains two types of shared resources:

## Organization Profile

[`profile/README.md`](profile/README.md) is displayed on the Waldron Lab's public GitHub organization page. It introduces the lab, its research focus (cancer genomics, microbiome profiling, and biostatistics), and community resources such as [BugSigDB](https://bugsigdb.org/) and the [Microbiome Virtual International Forum](https://microbiome-vif.org/).

## Setting up an R package repository

[`waldronlab/bioc-package-template`](https://github.com/waldronlab/bioc-package-template) has the two workflows a Waldron Lab R/Bioconductor package needs:

- `.github/workflows/ci.yml`: `R CMD check`, `BiocCheck` and test coverage (Codecov) in the Bioconductor container, from [Bioconductor/workflows](https://github.com/Bioconductor/workflows). It can also build a pkgdown site.
- `.github/workflows/claude.yml`: Claude Code responds to `@claude` and reviews each new pull request once (see [below](#claude-code-claudeyml-and-claude-reviewyml)).

**New package.** On the template's page, choose **Use this template → Create a new repository** with owner `waldronlab`, then follow "Setting up a new repository" in [its README](https://github.com/waldronlab/bioc-package-template#setting-up-a-new-repository).

**Existing repository**, or one created without the template. Copy the two workflow files into the repository's default branch:

```sh
mkdir -p .github/workflows
for f in ci.yml claude.yml; do
  curl -fsSL -o .github/workflows/$f \
    https://raw.githubusercontent.com/waldronlab/bioc-package-template/devel/.github/workflows/$f
done
```

Commit and push them directly, or merge them before testing Claude: the Claude action won't run on a pull request that changes its own workflow file. Nothing else is needed in `waldronlab`: the Claude GitHub App is installed on all repositories, and `CODECOV_TOKEN` and `CLAUDE_CODE_OAUTH_TOKEN` are organization secrets.

**pkgdown.** In `ci.yml`, set `enable_pkgdown: true` and uncomment the `permissions:` block, then set GitHub Pages to deploy from GitHub Actions and allow deployment from `RELEASE_*` branches ([steps](https://github.com/Bioconductor/workflows/blob/main/bioccheck_usage.md#repository-configuration-for-optional-features)). The site is built and deployed only on pushes to `RELEASE_*` branches, so a package that hasn't had a Bioconductor release yet gets no site until it does. See "Optional checks" in the [template README](https://github.com/waldronlab/bioc-package-template#optional-checks) for other settings.

## Reusable Workflows

Reusable GitHub Actions workflows live in [`.github/workflows/`](.github/workflows/) and can be called from any repository in the organization.

### `bioc-pr-cmdcheck-pkgdown.yml`

> **Deprecated.** Use the Bioconductor reusable workflows at [Bioconductor/workflows](https://github.com/Bioconductor/workflows) instead: copy its [`ci.yml`](https://github.com/Bioconductor/workflows/blob/main/ci.yml) template to `.github/workflows/ci.yml`, or build a customized one (Codecov, pkgdown) with the [workflow generator](https://bioconductor.github.io/workflows/). It runs `R CMD check` and `BiocCheck` in the official Bioconductor container, choosing the Bioconductor version from the branch name. It has no Docker Hub step; repositories that push a Docker image need a separate workflow for that. This workflow stays in place for repositories that still call it but will not be maintained.

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

### Claude Code: `claude.yml` and `claude-review.yml`

Two reusable workflows run [Claude Code](https://code.claude.com/docs/en/github-actions) on issues and pull requests:

| | `claude.yml` | `claude-review.yml` |
|---|---|---|
| Runs when | someone with write access mentions `@claude` or adds the `claude` label | a pull request is opened or marked ready for review |
| Does | whatever was asked: answers questions, reviews code, implements changes on a new `claude/...` branch | reviews the pull request and comments |
| Repository permissions | write | read, plus pull request comments |

They are separate so that the automatic review, which nobody asks for, runs with read-only access, and so that a repository can use `@claude` without automatic reviews.

#### Enable them in a repository

1. **Install the Claude GitHub App** on the repository: <https://github.com/apps/claude>. (It is installed on all `waldronlab` repositories.)
2. **Add this file** as `.github/workflows/claude.yml`. Delete the `review` job if you only want `@claude`.

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
     pull_request:
       types: [opened, ready_for_review]
   jobs:
     claude:
       if: github.event_name != 'pull_request'
       permissions:
         contents: write
         pull-requests: write
         issues: write
         id-token: write
         actions: read
       uses: waldronlab/.github/.github/workflows/claude.yml@devel
       secrets:
         CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
     review:
       if: github.event_name == 'pull_request'
       permissions:
         contents: read
         pull-requests: write
         id-token: write
       uses: waldronlab/.github/.github/workflows/claude-review.yml@devel
       secrets:
         CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
   ```

   The `permissions` blocks are required: a reusable workflow can't have more permissions than its caller, and the action needs `id-token: write` to get its GitHub App token. The `secrets:` blocks pass the `CLAUDE_CODE_OAUTH_TOKEN` secret by name. Don't shorten them to `secrets: inherit`: that only reaches a reusable workflow in the same organization, so it fails in repositories outside `waldronlab`.
3. **Merge it into the default branch before testing.** The action refuses to run on a pull request that adds or changes its own workflow file, and logs "Workflow validation failed" (the run still shows as successful).
4. **Test it** by commenting `@claude what does this repository do?` on any issue, and by opening a small pull request.

#### Authentication and usage

Runs use the organization secret `CLAUDE_CODE_OAUTH_TOKEN`, a Claude subscription token made with `claude setup-token`. They count against that subscription's usage limits and cost nothing extra.

The monthly API credit that comes with Claude Team and Max plans can't pay for these runs: it doesn't cover Claude Code, and the GitHub Action counts as Claude Code. Runs billed to that credit fail with "Credit balance too low".

The token acts as the person who created it. Anyone who can edit workflows in a repository that receives it can use it, so pass it only to repositories whose writers you trust. If it leaks or expires, make a new one with `claude setup-token` and replace the secret.

#### `claude.yml` inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `trigger_phrase` | `string` | `@claude` | Phrase in a comment, issue body or issue title that invokes Claude |
| `label_trigger` | `string` | `claude` | Issue label that invokes Claude. Use it instead of assigning the issue to Claude, which GitHub doesn't allow on private organization repositories. |
| `claude_args` | `string` | `""` | Extra Claude Code CLI arguments, e.g. `--max-turns 10` |
| `timeout_minutes` | `number` | `30` | Job timeout; also caps usage per run |
| `review_focus` | `string` | `""` | Repository-specific review instructions, used on pull requests. Pass the same text as to `claude-review.yml` so that `@claude review` follows them too |

#### `claude-review.yml` inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `review_focus` | `string` | `""` | Repository-specific instructions appended to the review prompt |
| `claude_args` | `string` | `""` | Extra Claude Code CLI arguments, e.g. `--model opus` |
| `timeout_minutes` | `number` | `20` | Job timeout |

The review runs once per pull request, not on every push, to save usage. Comment `@claude review` for another review. That review runs in `claude.yml`, which gives Claude the same review instructions and `review_focus` (if you pass it to both workflows) and tells it to read the changes with `gh pr diff`. The differences: it runs with write access, it posts inline comments but puts its summary in its progress comment rather than a separate comment, and it follows anything else the comment asks. The automatic review skips draft pull requests, pull requests opened by bots, and pull requests from forks, which can't receive secrets; someone with write access can still ask for a review of a fork pull request with `@claude review`.

Example inputs for an R package, in the caller file above:

```yaml
  claude:
    if: github.event_name != 'pull_request'
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    uses: waldronlab/.github/.github/workflows/claude.yml@devel
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    with:
      review_focus: "This is a Bioconductor package: check that exported functions are documented and that new code follows Bioconductor guidelines."
  review:
    if: github.event_name == 'pull_request'
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    uses: waldronlab/.github/.github/workflows/claude-review.yml@devel
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    with:
      review_focus: "This is a Bioconductor package: check that exported functions are documented and that new code follows Bioconductor guidelines."
```

#### Repositories outside the waldronlab organization

These workflows are public, so repositories owned by other GitHub accounts can call them too. Organization secrets don't reach them, so each such repository needs:

1. The Claude GitHub App installed on that account.
2. A repository secret `CLAUDE_CODE_OAUTH_TOKEN`, made with `claude setup-token` by the account owner.
3. The caller file above, unchanged. It passes the secret by name; with `secrets: inherit` instead, the run fails before starting with "Secret CLAUDE_CODE_OAUTH_TOKEN is required, but not provided while calling", because `secrets: inherit` only reaches reusable workflows in the caller's own organization.

Changes to these workflows affect every repository that calls them, including those outside the organization.

#### Notes

- Comments on issues and pull requests that don't mention `@claude` also start `claude.yml`, but the job is skipped. Skipped runs use no runner minutes; they do appear in the Actions tab.
- Don't add an `ANTHROPIC_API_KEY` secret or `anthropic_api_key` input: the action would use the API key instead of the subscription token.
