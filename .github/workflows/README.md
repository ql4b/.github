# Reusable workflows

Org-level reusable GitHub Actions workflows for ql4b repositories.

## `terraform-module-ci.yml`

The shared CI + release pipeline for ql4b Terraform modules, extracted from
`terraform-aws-ec2-proxy` (the lifecycle template). It runs:

- **Validate** (every PR and push): `terraform fmt`, `init`, `validate`,
  `tflint`, unit tests (`terraform test`), and `checkov`.
- **Update Docs** (push only): regenerates `README.md` with `terraform-docs`
  and commits it back.
- **Semantic Release** (push only): runs `semantic-release`.

A top-level `concurrency` group serializes runs per-ref so the docs/release
pushes of concurrent runs (e.g. a batch of Dependabot merges) don't collide on
a non-fast-forward push.

### Usage

Add a thin caller workflow to the module repo at
`.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    branches:
      - main
      - "+([0-9])?(.{+([0-9]),x}).x"
  pull_request:
    branches:
      - main
    types:
      - opened
      - synchronize
      - reopened

jobs:
  ci:
    uses: ql4b/.github/.github/workflows/terraform-module-ci.yml@main
    secrets: inherit
```

The caller owns the `on:` triggers; the reusable workflow still reads
`github.event_name` to gate the push-only jobs.

### Inputs

| Input | Default | Purpose |
|-------|---------|---------|
| `terraform_version` | `~> 1.7` | setup-terraform version constraint |
| `working_dir` | `.` | directory to run terraform/terraform-docs in |
| `checkov_skip` | `CKV_TF_1,CKV_AWS_382` | comma-separated Checkov check IDs to skip |
| `unit_test_filter` | `tests/unit.tftest.hcl` | `terraform test` filter; empty string skips unit tests |
| `run_docs` | `true` | run the terraform-docs auto-commit job on push |
| `run_release` | `true` | run semantic-release on push |

Override only what differs from the defaults:

```yaml
jobs:
  ci:
    uses: ql4b/.github/.github/workflows/terraform-module-ci.yml@main
    with:
      checkov_skip: "CKV_TF_1,CKV_AWS_382,CKV_AWS_341"
      unit_test_filter: ""   # module has no unit suite yet
    secrets: inherit
```

### Pinning

`@main` tracks the latest. To pin a module to an immutable revision, reference a
tag or commit SHA instead (e.g. `@v1`).
