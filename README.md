# github-actions

Reusable GitHub Actions workflows for Terraform modules in the `octo-devops-trainning` organization.

## Workflows

### `terraform-ci.yml`

Runs on pull requests. Exposes three parallel jobs:

| Job | What it does |
|---|---|
| `label-pr` | Ensures the PR has a version label (`patch`, `minor`, or `major`). Applies `patch` by default. |
| `static-validation` | Runs `make static-check` (fmt, validate, tflint, terraform-docs). |
| `terratest` | Starts LocalStack and runs `make terratest`. |

#### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `pr_number` | **yes** | — | PR number from the caller's event context. Required because the PR payload is not available inside a called workflow — see [Why `pr_number` is required](#why-pr_number-is-required). |
| `terraform_version` | no | `~1.5` | Terraform version constraint passed to `hashicorp/setup-terraform`. |
| `tflint_version` | no | `latest` | tflint version passed to `terraform-linters/setup-tflint`. |
| `terraform_docs_version` | no | `v0.18.0` | terraform-docs release tag to install. |
| `go_version_file` | no | `test/go.mod` | Path to the `go.mod` used to resolve the Go version for Terratest. |
| `localstack_image_tag` | no | `latest` | LocalStack Docker image tag. |
| `localstack_services` | no | `ec2` | Comma-separated list of LocalStack services to enable. |

#### Secrets

| Secret | Required | Description |
|---|---|---|
| `LOCALSTACK_AUTH_TOKEN` | no | Required only for LocalStack Pro. |

#### Usage

```yaml
jobs:
  ci:
    uses: octo-devops-trainning/github-actions/.github/workflows/terraform-ci.yml@main
    with:
      pr_number: ${{ github.event.pull_request.number }}
    permissions:
      pull-requests: write
      contents: read
    secrets: inherit
```

---

### `terraform-release.yml`

Runs on push to `main`. Creates a Git tag and GitHub Release with an auto-generated changelog. The version bump (`patch` / `minor` / `major`) is determined by the labels on the merged PR.

#### Inputs

None.

#### Usage

```yaml
jobs:
  release:
    uses: octo-devops-trainning/github-actions/.github/workflows/terraform-release.yml@main
    permissions:
      contents: write
      pull-requests: read
    secrets: inherit
```

---

## Makefile contract

Both workflows invoke `make` targets and **expect a `Makefile` to exist in the root of the calling repository** that implements the following targets:

| Target | Called by | What it must do |
|---|---|---|
| `static-check` | `static-validation` job | Run all static checks. Conventionally a composite of `fmt-check`, `validate`, `tflint`, and `docs-check`. Must exit non-zero on failure. |
| `terratest` | `terratest` job | Run the full Terratest suite. Conventionally `cd test && go test -v -timeout 30m ./...`. Must exit non-zero on failure. |

The workflows do not care about the internal implementation of these targets — only that they exist, are executable, and return a meaningful exit code. This makes it straightforward to adjust the checks in a specific module without touching the shared workflows.

A minimal reference `Makefile` for a Terraform module:

```makefile
.PHONY: fmt-check validate tflint docs-check static-check terratest

fmt-check:
	terraform fmt -check -recursive

validate:
	terraform init -backend=false && terraform validate

tflint:
	tflint --init && tflint --recursive

docs-check:
	terraform-docs --output-check .

static-check: fmt-check validate tflint docs-check

terratest:
	cd test && go test -v -timeout 30m ./...
```

---

## Why `pr_number` is required

When a reusable workflow runs via `workflow_call`, GitHub replaces `github.event` with the caller's inputs object. The original pull request payload — including labels and PR number — is no longer accessible inside the called workflow.

The caller captures `${{ github.event.pull_request.number }}` from its own event context and passes it as an input. The `label-pr` job then uses that number to call `github.rest.pulls.get()` and retrieve the current labels via the API.

---

## Versioning

Callers reference workflows at `@main` during initial setup. Once the workflows are stable, tag a release (e.g. `v1.0.0`) in this repository and update callers to pin to `@v1` for stability.
