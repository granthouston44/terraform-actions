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
Install the Go toolchain, optionally assume an AWS role, and execute Terratest suites using gotestsum with a concise summary.

```yaml
- uses: org/terraform-actions/.github/actions/run-terratest@v0
  with:
    go_version: 1.23.6
    test_pattern: all # or a regex like TestRoute53Module_BasicRecords
    timeout: 20m
    aws_role_to_assume: ${{ secrets.AWS_SANDBOX_ROLE }} # optional
    aws_region: eu-west-1
```

Notes:
- Runs in the job's working directory. Set `defaults.run.working-directory` to your `test` folder if needed.
- Uses `gotestsum` by default; no Terraform setup, fmt, validate, or tfsec here.
- Outputs `summary_path` and `report_path` can feed artifacts or downstream reporting steps.

### `.github/actions/terraform-plan-apply`
Deprecated: Prefer composing `terraform-setup` → `terraform-plan` and `terraform-apply` in workflows with stage gates.

### `.github/actions/terraform-plan`
Run terraform plan for a given environment and upload plan artifacts for cross-job reuse.

```yaml
- uses: org/terraform-actions/.github/actions/terraform-plan@v0
  with:
    environment: sandbox
```

### `.github/actions/terraform-apply`
Download the previously uploaded plan artifact and apply it for a given environment.

```yaml
- uses: org/terraform-actions/.github/actions/terraform-apply@v0
  with:
    environment: sandbox
```

## Reusable Workflow

### `.github/workflows/terraform-module.yml`
Recommended pattern: stage-gated workflows that compose `terraform-setup` → `terraform-plan` (artifact) → `terraform-apply`.

## Versioning

Tag git releases (for example `v0.1.0`) and reference them from consumer workflows with the corresponding ref. Each release should include a changelog entry summarising behaviour changes.

## Roadmap

- Publish reusable workflows that compose these actions for common Terraform module lifecycles.
- Add Trivy/Terratest reporting integrations once standardised across the platform.
