# Terraform Actions

Reusable GitHub composite actions maintained by the Tesco tooling team to standardise Terraform workflows across repositories.

## Provided Actions

### `.github/actions/terraform-setup`
Bootstrap Terraform repositories: configure private module access, install Terraform, run fmt/validate, and run tfsec (always on).

```yaml
- uses: actions/checkout@v4
- uses: org/terraform-actions/.github/actions/terraform-setup@v1
  with:
    terraform_version: 1.8.5 # optional; defaults to 1.8.5
```

Notes:
- Runs in the job's working directory. Set `defaults.run.working-directory` in your workflow or add `working-directory` on the step to target a module folder.

Key inputs:
- `terraform_version` – optional; version of Terraform CLI to install (default: 1.8.5).

### `.github/actions/run-terratest`
Install the Go toolchain and Terraform CLI, optionally assume an AWS role, and execute Terratest suites using gotestsum with a concise summary.

```yaml
- uses: org/terraform-actions/.github/actions/run-terratest@v1
  with:
    go_version: 1.23.6
    terraform_version: 1.8.5
    test_pattern: all # or a regex like TestRoute53Module_BasicRecords
    timeout: 20m
    aws_role_to_assume: ${{ secrets.AWS_SANDBOX_ROLE }} # optional
    aws_region: eu-west-1
```

Notes:
- Runs in the job's working directory. Set `defaults.run.working-directory` to your `test` folder if needed.
- Uses `gotestsum` by default; no fmt/validate/tfsec here (covered by `terraform-setup` in deploy workflows).
- Outputs `summary_path` and `report_path` can feed artifacts or downstream reporting steps.

Backend template rendering (required)
Place a template at `env/tfbackend.template` (or `env/<env>/tfbackend.template` for per‑env overrides) in your working directory. The `terraform-setup` action renders this to `env/<env>/<env>.tfbackend` before running `terraform init`.

Template variables:
- `${environment}` — expands to the workflow input `environment` passed to `terraform-setup`.
- Other variables are not set by default; you can still rely on `envsubst` to expand any variables you define in prior steps.

### `.github/actions/terraform-plan-apply`
Removed in v1.0.0. Prefer composing `terraform-setup` → `terraform-plan` and `terraform-apply` in workflows with stage gates.

### `.github/actions/terraform-plan`
Run terraform plan for a given environment and upload plan artifacts for cross-job reuse.

```yaml
- uses: org/terraform-actions/.github/actions/terraform-plan@v1
  with:
    environment: sandbox
```
Note: Ensure Terraform is installed first (use `terraform-setup`).

### `.github/actions/terraform-apply`
Download the previously uploaded plan artifact and apply it for a given environment.

```yaml
- uses: org/terraform-actions/.github/actions/terraform-apply@v1
  with:
    environment: sandbox
```
Note: Ensure Terraform is installed first (use `terraform-setup`).

## Reusable Workflow

### `.github/workflows/terraform-module.yml`
Recommended pattern: stage-gated workflows that compose `terraform-setup` → `terraform-plan` (artifact) → `terraform-apply`.

## Migration Guide

v1.0.0 (breaking changes):
- Removed `terraform-plan-apply` composite; use `terraform-setup` → `terraform-plan` → `terraform-apply`.
- `terraform-setup` simplified: always runs fmt/validate/tfsec; removed token/WD/toggles; relies on job working directory.
- `terraform-plan` and `terraform-apply` now assume Terraform is installed (run `terraform-setup` first); trimmed inputs.
- `run-terratest` simplified: standardized on `gotestsum`; removed Terraform setup and `working_directory` input; relies on job working directory.

Consumer updates:
- Update action refs from `@v0` to `@v1`.
- Replace usages of `terraform-plan-apply` with separate plan/apply steps.
- Insert `terraform-setup@v1` before plan/apply to install Terraform and run fmt/validate/tfsec.
- Set `defaults.run.working-directory` for infra/test paths instead of passing WD inputs.

## Versioning

Adopts semantic versioning with major tags (`v1`). Reference actions as `@v1` for stable major, or `@v1.0.0` for a fixed patch release.

## Roadmap

- Publish reusable workflows that compose these actions for common Terraform module lifecycles.
- Add Trivy/Terratest reporting integrations once standardised across the platform.
