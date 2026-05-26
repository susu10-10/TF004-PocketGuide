# Terraform CLI Appendix

This concise appendix contains useful `terraform` CLI commands, common flags, and CI examples you can paste into pipelines.

## Basic workflow

- Initialize a working directory:

```bash
terraform init
```

- Format and validate configuration:

```bash
terraform fmt -recursive
terraform validate
```

- Create a plan and save it for later apply:

```bash
terraform plan -out=tfplan
# Inspect the saved plan
terraform show -json tfplan | jq .
# Apply the saved plan
terraform apply "tfplan"
```

CI-friendly plan step (returns non-zero if changes):

```bash
terraform plan -detailed-exitcode -out=tfplan || PLAN_EXIT=$?
# Exit codes: 0 = no changes, 2 = changes, 1 = error
if [ "$PLAN_EXIT" -eq 2 ]; then
  echo "Changes detected"
fi
```

Formatting checks (useful in CI pre-commit):

```bash
# Fails if formatting is required
terraform fmt -check -recursive
```

## Scripting and automation

- Produce machine-readable output for automation:

```bash
terraform show -json > state.json
```

- Extract values with `jq` (example: list s3 bucket names):

```bash
jq -r '.values.root_module.resources[] | select(.type=="aws_s3_bucket") | .values.bucket' state.json
```

## State inspection and manipulation

- List resources in state:

```bash
terraform state list
```

- Show a single resource from state:

```bash
terraform state show module.vpc.aws_vpc.main
```

- Move resources between addresses (useful for refactors):

```bash
terraform state mv old.address new.address
```

- Remove a resource from state (do not destroy in cloud):

```bash
terraform state rm aws_s3_bucket.legacy_assets
```

## Importing

- CLI import (legacy) — binds existing resource to config address:

```bash
terraform import aws_s3_bucket.legacy_assets my-company-legacy-assets
```

- New `import` block (1.5+) is preferred for declarative imports.

## Quick recovery tips

- Backup state before risky operations:

```bash
terraform state pull > state-backup-$(date +%F).json
```

- If you accidentally pushed an invalid state, re-upload a known-good backup with caution:

```bash
terraform state push --force state-backup.json
```

## Provider and workspace

- Show provider versions and lockfile info:

```bash
cat .terraform.lock.hcl
```

- Manage workspaces (CLI):

```bash
terraform workspace list
terraform workspace select staging
terraform workspace new feature-x
```

Provider authentication (examples):

- For AWS, prefer dynamic credentials or environment variables over hardcoded keys:

```bash
export AWS_PROFILE=myprofile
export AWS_REGION=us-east-1
```

- For Terraform Cloud/HCP remote runs, use `terraform login` to configure credentials.

