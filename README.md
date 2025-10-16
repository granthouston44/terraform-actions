# Terraform Actions

Reusable GitHub composite actions maintained by the Tesco tooling team to standardise Terraform workflows across repositories.

## Provided Actions

### `.github/actions/terraform-setup`
Bootstrap Terraform repositories: configure private module access, install Terraform, run fmt/validate, and run tfsec (always on).

```yaml
- uses: actions/checkout@v4
- uses: org/terraform-actions/.github/actions/terraform-setup@v0
  with:
    terraform_version: 1.8.5 # optional; defaults to 1.8.5
```

Notes:
- Runs in the job's working directory. Set `defaults.run.working-directory` in your workflow or add `working-directory` on the step to target a module folder.

Key inputs:
- `terraform_version` – optional; version of Terraform CLI to install (default: 1.8.5).

### `.github/actions/run-terratest`
Install the Go toolchain, optionally assume an AWS role, and execute Terratest suites with summarised output.

```yaml
- uses: org/terraform-actions/.github/actions/run-terratest@v0
  with:
    go_version: 1.23.6
    working_directory: test
    test_pattern: TestRoute53Module_BasicRecords
    aws_role_to_assume: ${{ secrets.AWS_SANDBOX_ROLE }}
    aws_region: eu-west-1
```

Notes:
- The target repository must contain the `cmd/gotestfmt` formatter (or adjust the action to match your formatter).
- Outputs `summary_path` and `report_path` can feed artifacts or downstream reporting steps.

### `.github/actions/terraform-plan-apply`
Handle backend-aware init, change-aware planning, optional JSON exports, and gated apply steps.

```yaml
- uses: org/terraform-actions/.github/actions/terraform-plan-apply@v0
  with:
    working_directory: infra
    backend_config_file: env/sandbox.backend.hcl
    var_file: env/sandbox.tfvars
    plan_file: sandbox.tfplan
    run_plan: true
    run_apply: false
    aws_role_to_assume: ${{ secrets.AWS_SANDBOX_ROLE }}
```

Outputs include `plan_status`, `plan_exit_code`, and paths to the rendered plan artifacts for use in comments or uploads. Apply is skipped unless `run_apply: true` and can be paired with required environment approvals.

### `.github/actions/terraform-plan`
A thin wrapper around the plan portion of `terraform-plan-apply` that also uploads the plan artifacts for cross-job reuse.
Note: Assumes Terraform is already installed (run `terraform-setup` earlier in the job).

```yaml
- uses: org/terraform-actions/.github/actions/terraform-plan@v0
  with:
    working_directory: infra
    backend_config_file: env/sandbox.backend.hcl
    var_file: env/sandbox.tfvars
    plan_file: sandbox.tfplan
    artifact_name: tfplan
```

### `.github/actions/terraform-apply`
Downloads a previously uploaded plan artifact and performs an apply, delegating to `terraform-plan-apply` under the hood.
Note: Assumes Terraform is already installed (run `terraform-setup` earlier in the job).

```yaml
- uses: org/terraform-actions/.github/actions/terraform-apply@v0
  with:
    working_directory: infra
    backend_config_file: env/sandbox.backend.hcl
    var_file: env/sandbox.tfvars
    plan_file: sandbox.tfplan
    artifact_name: tfplan
```

## Reusable Workflow

### `.github/workflows/terraform-module.yml`
Orchestrates a standard module lifecycle: Terratest → Plan (PR comment) → gated Apply. Intended to be called from consumer repositories with environment protections.

```yaml
name: Terraform CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  module:
    uses: org/terraform-actions/.github/workflows/terraform-module.yml@v0
    with:
      working_directory: infra
      backend_config_file: env/sandbox.backend.hcl
      var_file: env/sandbox.tfvars
      environment: prod
    secrets: inherit
```

## Versioning

Tag git releases (for example `v0.1.0`) and reference them from consumer workflows with the corresponding ref. Each release should include a changelog entry summarising behaviour changes.

## Roadmap

- Publish reusable workflows that compose these actions for common Terraform module lifecycles.
- Add Trivy/Terratest reporting integrations once standardised across the platform.
