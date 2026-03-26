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
    uses: waldronlab/.github/.github/workflows/bioc-pr-cmdcheck-pkgdown.yml@main
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
