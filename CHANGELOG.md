# Changelog

All notable changes to this repository are documented here.

## v1.0.0

Breaking release that simplifies actions and removes monolithic composite.

- Removed `.github/actions/terraform-plan-apply`; compose `terraform-setup` → `terraform-plan` → `terraform-apply` instead.
- Simplified `.github/actions/terraform-setup`:
  - Always configures Git with `github.token` for private modules
  - Always runs `terraform fmt -check -diff`, local `init`, and `validate`
  - Always runs `tfsec` with `--soft-fail --minimum-severity high`
  - Removed inputs: working_directory, github_token, configure_private_modules, run_fmt, run_local_init, run_validate, tfsec_*
  - Relies on the job working directory
- Simplified `.github/actions/terraform-plan` and `.github/actions/terraform-apply`:
  - Removed Terraform installation from composites; assume Terraform is installed via `terraform-setup`
  - Inlined backend/plan path computation (apply)
- Simplified `.github/actions/run-terratest`:
  - Standardized on `gotestsum`; removed formatter toggles and `gotestfmt`
  - Removed Terraform setup and `working_directory` input; relies on job working directory
  - Minimal inputs: `go_version`, `test_pattern`, `timeout`, optional AWS role/region/session

## v0.x

Initial iterations of the composites and reusable workflow.
