# Terraform Actions

Reusable GitHub composite actions maintained by the Tesco tooling team to standardise Terraform workflows across repositories.

## Composite Actions

### `.github/actions/terraform-setup`
Bootstrap Terraform repositories: configure private module access, install Terraform, run fmt/validate, and run tfsec (always on).

```yaml
- uses: actions/checkout@v4
- uses: tescotestims/terraform-actions/.github/actions/terraform-setup@v1
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
- uses: tescotestims/terraform-actions/.github/actions/run-terratest@v1
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

### `.github/actions/print-run-context`
Append a human‑readable summary (title, repo/run links, env, Terraform/Terratest metadata) to the job summary.

- Inputs: `title`, `environment`, `aws_account_id`, `aws_role_arn`, `working_directory`, `backend_config_file`, `var_file`, `plan_file`, `terraform_version`, `go_version`, `terratest_*`, `notes`
- Example:
```yaml
- uses: tescotestims/terraform-actions/.github/actions/print-run-context@v1
  with:
    title: Terraform Module Workflow Context
    environment: sandbox
    aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
    working_directory: infra
    plan_file: plan-sandbox.tfplan
    terraform_version: 1.8.5
```

### `.github/actions/terraform-plan`
Run terraform plan for a given environment and upload plan artifacts for cross-job reuse.

```yaml
- uses: tescotestims/terraform-actions/.github/actions/terraform-plan@v1
  with:
    environment: sandbox
```
Note: Ensure Terraform is installed first (use `terraform-setup`).

### `.github/actions/terraform-plan-destroy`
Run terraform plan -destroy for a given environment and upload plan artifacts for cross-job reuse.

```yaml
- uses: tescotestims/terraform-actions/.github/actions/terraform-plan-destroy@v1
  with:
    environment: sandbox
```
Note: Ensure Terraform is installed first (use `terraform-setup`).

### `.github/actions/terraform-apply`
Apply the configuration for a given environment (file-driven backend + tfvars). Assumes `terraform-setup` has run.

```yaml
- uses: tescotestims/terraform-actions/.github/actions/terraform-apply@v1
  with:
    environment: sandbox
```
Note: Ensure Terraform is installed first (use `terraform-setup`).

### `.github/actions/terraform-destroy`
Destroy the configuration for a given environment (file-driven backend + tfvars). Assumes `terraform-setup` has run.

```yaml
- uses: tescotestims/terraform-actions/.github/actions/terraform-destroy@v1
  with:
    environment: sandbox
```

## Reusable Workflows

### `.github/workflows/terraform-module.yml`
Single reusable workflow for `plan`, `apply`, and `destroy` (via `action` input`). Uses `<env>-deploy` for approval gates and `<env>` for execution.

- Inputs: `working_directory`, `environment`, `action` (plan|apply|destroy)
- Behavior:
  - plan: setup → plan (artifacts + summary)
  - apply: setup → plan → approval in `<env>-deploy` → setup → apply
  - destroy: setup → plan-destroy (artifacts + summary) → approval in `<env>-deploy` → setup → destroy
- Example:
```yaml
jobs:
  module:
    uses: tescotestims/terraform-actions/.github/workflows/terraform-module.yml@v1
    with:
      working_directory: infra
      environment: sandbox
      action: apply
    secrets: inherit
```

### `.github/workflows/terratest.yml`
Reusable Terratest workflow. Sets up Go + Terraform, optionally assumes a role, runs tests via gotestsum, publishes summary + artifacts.

- Inputs: `go_version`, `terraform_version`, `working_directory` (default `test`), `test_pattern` (default `all`), `timeout`, `environment`
- Example:
```yaml
jobs:
  terratest:
    uses: tescotestims/terraform-actions/.github/workflows/terratest.yml@v1
    with:
      go_version: 1.23.6
      terraform_version: 1.8.5
      working_directory: test
      test_pattern: all
      environment: sandbox
    secrets: inherit
```

### `.github/workflows/aws-account-bootstrap.yml`
Opinionated workflow to plan/apply bootstrap roots behind an approval gate.

- Inputs: `working_directory` (default `.`), `environment`, `apply_auto_approve` (bool)


## Requirements

- Org settings: allow reuse of workflows from private repositories (for cross‑repo reuse)
- Repo vars: `AWS_ACCOUNT_ID` (or equivalent)
- Environments:
  - `<env>` (e.g. `sandbox`) — unprotected; holds env‑scoped vars/secrets; used by plan/apply/destroy
  - `<env>-deploy` — protected; Required reviewers or Wait timer; used only for approval gates
- IAM OIDC trust: allow `repo:<org>/<repo>:environment:<env>` in the role trust policy

## Versioning

Use `@v1` for stable major, or `@main` while iterating.
