# Objective 7: Maintain infrastructure with Terraform

Now that you deeply understand state, it's time to master the operational lifecycle. This objective covers three key areas that every Terraform practitioner needs in production:

Below is a breakdown of the sub-objectives we will cover in this section: Feel free to jump to any section directly, but I recommend going through them in order for the best learning experience.

- [**Objective 7a:** Import existing infrastructure into your Terraform workspace](#objective-7a-import-existing-infrastructure-into-your-terraform-workspace)

- [**Objective 7b:** Use the CLI to inspect state](#objective-7b-use-the-cli-to-inspect-state)

- [**Objective 7c:** Describe when and how to use verbose logging](#objective-7c-describe-when-and-how-to-use-verbose-logging)

- [**Objective 7: Quiz**](#objective-7-quiz)

We'll examine each sub-objective with the full code explanations you've asked for, and then test your understanding with a comprehensive gauntlet.

---

## Objective 7a: Import existing infrastructure into your Terraform workspace

*Source: [`Import Terraform configuration`](https://developer.hashicorp.com/terraform/tutorials/state/state-import), [`Import an existing resource`](https://developer.hashicorp.com/terraform/cli/v1.12.x/import/usage), [`Command: import`](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/import)*

### 7a.1: The Problem — Brownfield Deployments and the Need for Import

Most organizations don't start with a greenfield Terraform deployment. They have existing infrastructure `EC2 instances`, `S3 buckets`, `databases`—that was created manually through the AWS Console, CLI scripts, or other tools. Terraform knows nothing about these resources.

Import is the process of bringing this existing "brownfield" infrastructure under Terraform management without destroying and recreating it.

**What import does:** It writes the resource's real-world ID and attributes into your Terraform state file, creating a binding between a `resource` block in your configuration and an existing cloud resource. From that point forward, Terraform manages the resource as if it had created it.

### 7a.2: The Configuration-Driven Import (Terraform 1.5+)

The modern, recommended approach uses the `import` block. It's declarative, version-controlled, and integrates with the plan/apply workflow.

**Step 1: Write the destination resource block.**

You must define the resource in your configuration **before** importing it. At minimum, the block needs the required arguments for that resource type. You don't need all the fancy settings—just enough that Terraform can plan against it after import.

```hcl
# We'll import an existing S3 bucket that was created manually.
# This is the "destination" resource that will represent it in Terraform.

resource "aws_s3_bucket" "legacy_assets" {
  # 'bucket' is a required argument for aws_s3_bucket.
  # The value MUST exactly match the existing bucket's name.
  bucket = "my-company-legacy-assets"
}
```

**Explanation:**

- `resource "aws_s3_bucket" "legacy_assets" {` — Declares the resource block. `aws_s3_bucket` is the resource type from the AWS provider. `legacy_assets` is the local name you're giving it in Terraform. This local name can be anything you want; it doesn't need to match the real bucket name.
- `bucket = "my-company-legacy-assets"` — This is a required argument that tells the AWS provider which bucket to manage. The value must be exactly the name of the existing S3 bucket in AWS.

**Step 2: Write the `import` block.**

```hcl
# The import block tells Terraform: "There's already an S3 bucket
# with this ID in AWS. Please associate it with the resource block
# I just defined so Terraform can start managing it."

import {
  # 'id' is the cloud provider's unique identifier for the resource.
  # For an S3 bucket, the identifier is simply the bucket name.
  id = "my-company-legacy-assets"

  # 'to' is the Terraform resource address to import into.
  # It follows the format: RESOURCE_TYPE.LOCAL_NAME
  to = aws_s3_bucket.legacy_assets
}
```

**Explanation:**

- `import {` — Declares an import block. You can have multiple `import` blocks per configuration, one for each resource you're importing.
- `id = "my-company-legacy-assets"` — The identifier that the cloud provider uses to uniquely locate this resource. For `aws_s3_bucket`, the identifier is the bucket name. For `aws_instance`, it would be the instance ID (`i-12345`). For `aws_vpc`, it's the VPC ID (`vpc-abc123`). This is provider-specific; always check the provider documentation for the correct ID format.
- `to = aws_s3_bucket.legacy_assets` — The destination resource address. This must exactly match the resource type and local name you defined in Step 1. Terraform will bind the existing resource to this address in the state file.

**Step 3: Run `terraform plan`.**

```bash
terraform plan
```

Terraform will read the `import` block, query AWS for the bucket's current attributes, and show you that it will import the resource. It will also show any configuration attributes it will update after import (e.g., if your `resource` block has additional settings that differ from the actual bucket).

**Step 4: Optionally generate configuration.**

If you're importing a complex resource and don't know all its current attributes, you can ask Terraform to generate the full configuration for you:

```bash
terraform plan -generate-config-out=generated.tf
```

This creates a file `generated.tf` with all the resource's arguments filled in based on its actual state. You can then review this file, keep what you need, remove what you don't, and move the relevant parts into your main configuration.

**Step 5: Run `terraform apply`.**

```bash
terraform apply
```

Terraform imports the resource into the state file. From this point forward, Terraform manages the bucket as if it created it.

**Step 6: (Optional) Remove the `import` block.**

Once the import is complete and the resource is in state, the `import` block has done its job. You can remove it or keep it as a historical record of where the resource came from. The block is idempotent—if you run `apply` again with the block still present, Terraform sees the resource is already in state and does nothing.

---

### 7a.3: Importing Resources with `for_each` or `count`

You can import multiple instances of a resource using `for_each` or `count` in the import block.

```hcl
# Import multiple EC2 instances using for_each.
# Assume we have three instances: i-abc, i-def, i-ghi.

resource "aws_instance" "legacy_servers" {
  for_each = {
    "web"   = "i-abc123"
    "api"   = "i-def456"
    "worker" = "i-ghi789"
  }

  ami           = "ami-12345"
  instance_type = "t3.micro"
}

import {
  for_each = {
    "web"   = "i-abc123"
    "api"   = "i-def456"
    "worker" = "i-ghi789"
  }

  id = each.value       # The instance ID from the map
  to = aws_instance.legacy_servers[each.key]  # The resource instance address
}
```

**Explanation:**

- `for_each = { ... }` — Defines a map where keys are the Terraform instance keys (`web`, `api`, `worker`) and values are the cloud identifiers (`i-abc123`, etc.).
- `id = each.value` — For each iteration, sets the import ID to the cloud resource identifier.
- `to = aws_instance.legacy_servers[each.key]` — Specifies the exact resource instance address using the `each.key` as the index.

---

### 7a.4: The Legacy `terraform import` Command (Pre-1.5)

Before the `import` block existed, you had to use the CLI command:

```bash
terraform import aws_s3_bucket.legacy_assets my-company-legacy-assets
```

**Format:** `terraform import RESOURCE_ADDRESS CLOUD_ID`

This command directly writes the resource into the state file. It does **not** generate or validate any configuration. You must manually write the matching `resource` block afterward. There is no plan step, no review, it's immediate and irreversible on the state file.

**The exam may still reference `terraform import` command syntax, but the `import` block is the modern best practice.**

---

### 7a.5: Important Limitations of Import

The official study guide lists these critical limitations:

1. **Import reads the current state reported by the provider.** It cannot determine the intent, health, or history of the resource.
2. **Import does not generate resource relationships.** If your S3 bucket has lifecycle policies, event notifications, or depends on other resources, the `import` block doesn't capture these dependencies. You must manually add those relationships in your configuration.
3. **Import does not detect which arguments you can omit.** The generated configuration includes every possible argument, including defaults and empty values. You must prune the generated configuration.
4. **Not all resources support import.** Check the provider documentation for each resource type.
5. **Import does not guarantee Terraform can manage all aspects of the resource.** Some resources created externally may have configurations that Terraform cannot update without recreation.

---

### 7a.6: Hands-On Lab — Configuration-Driven Import with Local Provider

We'll simulate importing an existing file that was created outside Terraform.

**Directory:** `obj7a-lab`

**Step 1: Create a file outside Terraform.**

```bash
# Simulate manually creating a "cloud resource"
echo "This file was created manually, outside Terraform." > manual-resource.txt
```

**Step 2: Write the Terraform configuration.**
```hcl
# main.tf
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

# Define the resource we want to import the existing file into.
# The 'filename' argument must match the existing file's path.
resource "local_file" "imported_file" {
  filename = "${path.module}/manual-resource.txt"
  # 'content' will be overwritten to match the current file content after import
}

# The import block tells Terraform to find the existing file and bind it.
import {
  # For the local provider, the file's ID is its SHA256 checksum.
  # We provide the file path as the identifier—the provider knows how to resolve it.
  id = "${path.module}/manual-resource.txt"
  to = local_file.imported_file
}
```

**Line-by-line:**

- `import {` — Declares the import block for this resource.
- `id = "${path.module}/manual-resource.txt"` — The identifier. For `local_file`, the `id` is the SHA256 hash of the file content. However, the import process accepts the file path as the identifier and computes the hash internally. `${path.module}` is a built-in value that evaluates to the absolute path of the directory containing the module where this expression appears.
- `to = local_file.imported_file` — Ties the import to the `resource "local_file" "imported_file"` block we defined.

**Step 3: Run the import workflow.**

```bash
terraform init
terraform plan
# Observe that Terraform will import the file and then show that 'content' will be set to the actual file content.

terraform apply
# Terraform imports the file into state. The file content in state will match the file on disk.
```

**Step 4: Verify.**
```bash
terraform state show local_file.imported_file
# Shows the resource with the imported attributes.
```

**Step 5: Clean up.**
```bash
# Optionally remove the import block from main.tf (it's idempotent, but clean code is better)
# Now Terraform manages the file. Any changes you make to the file will be detected.
```
---

### 7a.7: Mini-Quiz for 7a

1. True/False: After running `terraform import aws_instance.web i-12345`, the matching `resource` block is automatically generated in your configuration.
2. Multiple Choice: In an `import` block, what does the `id` argument represent?
   A. The Terraform resource address.
   B. The local name of the resource in the configuration.
   C. The cloud provider's unique identifier for the existing resource.
   D. The state file serial number.
3. Multiple Answer: Which of the following are limitations of `terraform import`? (Select TWO)
   A. It cannot import resources that use `for_each`.
   B. It does not generate resource dependency relationships.
   C. It cannot import resources created by another IaC tool.
   D. Not all resource types support import.

<details>
<summary>Show Answers</summary>

1. False - The `terraform import` command does not generate any configuration. You must manually write the matching `resource` block after the import. The `import` block in Terraform 1.5+ is the modern way to handle imports, but it also requires you to define the resource block first.
2. C - The `id` argument in an `import` block is the cloud provider's unique identifier for the existing resource. This is how Terraform locates the resource to import it into state.
3. B, D - `terraform import` does not generate resource dependency relationships (B), and not all resource types support import (D). `terraform import` can be used with resources that use `for_each` or `count` if you structure the import correctly.

</details>


## Objective 7b: Use the CLI to inspect state

*Source: [`Manage resources in Terraform state`](https://developer.hashicorp.com/terraform/tutorials/state/state-cli), [`terraform state command`](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/state)*

### 7b.1: State Inspection Commands Overview

The Terraform CLI provides a suite of subcommands under `terraform state` for inspecting and manipulating the state file without directly editing the JSON. These commands are your primary debugging tools.

### 7b.2: `terraform state list`

Lists all resource addresses in the state file.

```bash
terraform state list
```
**Output example:**
```
random_pet.server
local_file.config
module.vpc.aws_vpc.main
module.vpc.aws_subnet.public[0]
module.vpc.aws_subnet.public[1]
data.aws_ami.ubuntu
```

**Key points:**

- Resources are listed in module depth order, then alphabetically.
- Resources with `count` or `for_each` show their instance index (`[0]`, `[1]`, `["key"]`).
- Data sources that are cached in state also appear (prefixed with `data.`).
- Modules are prefixed with `module.MODULE_NAME.`

**Filtering:**
```bash
# List only resources in the vpc module
terraform state list module.vpc.*

# List only aws_instance resources
terraform state list aws_instance.*

# List a specific named resource
terraform state list aws_instance.web
```
---

### 7b.3: `terraform state show`

Displays the full attributes of a single resource from the state file.

```bash
terraform state show local_file.config
```

**Output example:**
```
# local_file.config:
resource "local_file" "config" {
    content              = "Hello, Terraform!"
    content_base64sha256 = "abc123..."
    content_base64sha512 = "def456..."
    content_md5          = "ghi789..."
    content_sha1         = "jkl012..."
    content_sha256       = "mno345..."
    content_sha512       = "pqr678..."
    directory_permission = "0777"
    file_permission      = "0777"
    filename             = "./config.txt"
    id                   = "abc123def456ghi789"
}
```

**Use cases:**

- Quickly check the current value of a resource attribute without opening the JSON.
- Verify that the state reflects what you expect after an import.
- Debug why a resource is being recreated: check if its computed attributes differ from what's in state.

### 7b.4: `terraform show`

Display the entire state in human-readable format (or JSON with `-json`).

```bash
# Human-readable output of the entire state
terraform show

# JSON output for scripting
terraform show -json
```

**JSON output is useful for:**

- Integrating with external tools.
- Parsing resource attributes programmatically.
- Comparing state between environments.

### 7b.4a: Scripting with `terraform show -json`

`terraform show -json` produces a machine-friendly JSON representation of the state which is useful for automation. Example: extract all `aws_s3_bucket` names with `jq`:

```bash
terraform show -json | jq -r '.values.root_module.resources[] | select(.type=="aws_s3_bucket") | .values.bucket'
```

Note: the JSON schema may change between Terraform versions; pin your Terraform version in CI when relying on this output.

### 7b.5: `terraform state pull` and `terraform state push`

These commands interact directly with the configured backend to download or upload state.

```bash
# Download the current state from the remote backend and output as JSON to stdout
terraform state pull > current-state.json
```

```bash
# Upload a state file to the remote backend (EXTREMELY DANGEROUS)
terraform state push current-state.json
```

**Warnings from the study guide:**

- `terraform state push` will overwrite the remote state. Terraform protects against accidental push by checking the `lineage` and `serial` values.
- If the `serial` in the pushed state is lower than the current remote state, Terraform rejects the push (because the remote state has changed since you pulled).
- Use `-force` to bypass these protections only if you are absolutely certain.
- Always back up the remote state before pushing.

---

### 7b.6: Hands-On Lab — State Inspection

We'll use the state file from any previous lab (e.g., the `obj6d-lab` or create a quick one).

**Directory:** `obj7b-lab`

**File: `main.tf`**
```hcl
terraform {
  required_providers {
    random = { source = "hashicorp/random", version = "~> 3.6" }
    local  = { source = "hashicorp/local", version = "~> 2.5" }
  }
}

resource "random_pet" "demo" {
  length = 2
}

resource "local_file" "log" {
  filename = "${path.module}/${random_pet.demo.id}.log"
  content  = "Log for ${random_pet.demo.id}"
}
```

**Steps:**

1. `terraform init && terraform apply -auto-approve`
2. Run `terraform state list` to see both resources.
3. Run `terraform state show random_pet.demo` to see its attributes.
4. Run `terraform show` to see the full state in human-readable format.
5. Run `terraform state pull > state-backup.json` to save a copy of the state.
6. Run `terraform state show local_file.log` to see the filename and content.

### 7b.7: Mini-Quiz for 7b

1. True/False: `terraform state list` shows only managed resources; data sources are excluded.
2. Multiple Choice: Which command would you use to see the detailed attributes of a specific resource from the state file?
   A. `terraform plan`
   B. `terraform state list`
   C. `terraform state show`
   D. `terraform output`
3. True/False: `terraform state push` will unconditionally overwrite the remote state without any safety checks.

<details>
<summary>Show Answers</summary>

1. False - `terraform state list` will include cached data sources if they are present in the state; it is not limited to only managed resources.
2. C - `terraform state show` displays detailed attributes for a single resource from the state file.
3. False - `terraform state push` has safety checks (it validates `lineage` and `serial`) and will not unconditionally overwrite the remote state without explicit force.

</details>

## Objective 7c: Describe when and how to use verbose logging

*Source: [`Debugging Terraform`](https://developer.hashicorp.com/terraform/internals/v1.12.x/debugging), [`Enable Terraform logs`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/troubleshooting-workflow#enable-terraform-logging)*

### 7c.1: The Problem — When Terraform Fails, You Need More Information

When Terraform behaves unexpectedly, a plan shows changes it shouldn't, an apply fails with a cryptic error, a provider returns an API error, you need more information than the CLI provides by default. Terraform has a comprehensive logging system that can be enabled via environment variables.

### 7c.2: Core Terraform Logging (`TF_LOG`)

The primary environment variable is `TF_LOG`. It controls the verbosity of Terraform's own logs.

```bash
# Enable Terraform core logging at DEBUG level
export TF_LOG=DEBUG
```

**Log Levels (in order of decreasing verbosity):**

| Level | Description |
|-------|-------------|
| `TRACE` | Most verbose. Shows every internal operation, graph walk, RPC call. |
| `DEBUG` | Detailed operational information. Useful for debugging configuration behavior. |
| `INFO` | General operational messages. Good for understanding high-level flow. |
| `WARN` | Warning messages that don't stop operation. |
| `ERROR` | Only error messages that cause operation failure. |

**Setting the log path:**
```bash
export TF_LOG_PATH=terraform.log
```
This redirects all logs to a file instead of `stderr`. The file will be created or appended in the current working directory.

**Example session:**
```bash
export TF_LOG=TRACE
export TF_LOG_PATH=debug-apply.log
terraform apply
# After completion, review debug-apply.log for detailed information
```

### 7c.3: Provider-Specific Logging

You can enable logging only for provider plugins, separate from Terraform Core:

```bash
# Only log provider operations, not core Terraform internals
export TF_LOG_PROVIDER=TRACE
```

This is useful when you suspect a bug in a specific provider but want to avoid the noise from core logging.

You can also target specific providers by combining `TF_LOG_PROVIDER` with a specific provider log level:

```bash
# Only debug the AWS provider
export TF_LOG=INFO                      # Core at INFO level
export TF_LOG_PROVIDER_AWS=TRACE        # AWS provider at TRACE level
```
---

### 7c.4: When to Use Logging

**Production debugging:**

- An `apply` fails with an API error from a cloud provider. Enable `TF_LOG=DEBUG` and re-run to see the exact API request and response.
- A plan shows unexpected resource recreation. Enable `TF_LOG=TRACE` to trace the dependency graph and state refresh logic.

**Bug reporting:**

- When filing a GitHub issue for Terraform Core or a provider, always include `TRACE`-level logs.
- The study guide recommends using `TF_LOG=TRACE` for bug reports because it provides the highest level of detail for developers.

**CI/CD pipelines:**

- Set `TF_LOG=WARN` for routine runs to catch any unexpected behaviors without overwhelming the log output.
- Set `TF_LOG_PATH` to capture logs for post-mortem analysis.

### 7c.5: Logging Format

By default, logs are formatted as human-readable text lines. For machine consumption, set:

```bash
export TF_LOG=JSON
```

This produces JSON-encoded log lines at the `TRACE` level or higher, suitable for parsing by log aggregation tools.

**Warning from the study guide:** The JSON encoding format is **not a stable interface**. It may change between Terraform versions. Only use it for tooling that you maintain and update alongside Terraform.

### 7c.6: Other Debugging Environment Variables

- `TF_LOG_CORE`: Enables logging only for Terraform Core (the orchestration layer).
- `TF_LOG_PROVIDER`: Enables logging only for all providers.
- `TF_LOG_PROVIDER_<PROVIDER_NAME>`: Enables logging for a specific provider (e.g., `TF_LOG_PROVIDER_AWS`).

---

### 7c.7: Hands-On Lab — Enable Logging

We'll enable logging for a simple operation and inspect the output.

**Directory:** `obj7c-lab`

**Step 1: Create a simple configuration.**
```hcl
# main.tf
resource "local_file" "test" {
  filename = "${path.module}/log-test.txt"
  content  = "Testing logging."
}
```

**Step 2: Enable logging and run apply.**
```bash
terraform init

# Set environment variables for verbose logging
export TF_LOG=TRACE
export TF_LOG_PATH=apply-trace.log

terraform apply -auto-approve
```

**Step 3: Inspect the log file.**
```bash
cat apply-trace.log | head -50
```

You'll see detailed entries about:
- Provider initialization
- State file reading
- Plan generation
- Resource creation
- State writing

**Step 4: Disable logging.**
```bash
unset TF_LOG
unset TF_LOG_PATH
```

### 7c.8: Mini-Quiz for 7c

1. True/False: Setting `TF_LOG=INFO` provides more verbose output than `TF_LOG=TRACE`.
2. Multiple Choice: Which environment variable would you set to capture Terraform logs to a file instead of the terminal?
   A. `TF_LOG_FILE`
   B. `TF_LOG_PATH`
   C. `TF_DEBUG_PATH`
   D. `TF_LOG_OUTPUT`
3. Multiple Answer: Which of the following are valid `TF_LOG` levels? (Select THREE)
   A. `TRACE`
   B. `DEBUG`
   C. `VERBOSE`
   D. `WARN`
   E. `ERROR`


<details>
<summary>Show Answers</summary>
1. False - `TF_LOG=TRACE` is more verbose than `TF_LOG=INFO`. The order of verbosity from most to least is: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`.
2. B - `TF_LOG_PATH` is the environment variable used to specify a file path for Terraform logs. Setting this variable directs all logs to the specified file instead of `stderr`.
3. A, B, D - Valid `TF_LOG` levels include `TRACE`, `DEBUG`, `INFO`, `WARN`, and `ERROR`. For this question (choose three), `TRACE`, `DEBUG`, and `WARN` are valid examples. `VERBOSE` is not valid.

</details>

## 7b.8: Refactor state — move, rename, and `moved` blocks

Sometimes you need to rename resources, move them between modules, or change the logical addressing without recreating the real-world resource. Terraform supports safe refactoring via CLI commands and configuration helpers.

- `terraform state mv OLD_ADDR NEW_ADDR` — Move a resource or an instance within the state file. This does not change the real resource, only the state mapping.

Example: move a resource into a module after refactoring your configuration:

```bash
# Move aws_instance.web to module.web.aws_instance.web
terraform state mv aws_instance.web module.web.aws_instance.web
```

- `terraform state rm ADDR` — Remove a resource from state without destroying the underlying resource (useful when you intentionally want Terraform to stop managing it).

- `moved` block in configuration (Terraform 1.4+) — Declare stable migrations so Terraform can automatically rewrite state during `terraform init`/`apply` without manual CLI state moves. Use `moved` to record renames safely in VCS.

Example `moved` block:

```hcl
# In your root module
moved {
  from = aws_instance.old_name
  to   = module.web.aws_instance.web
}
```

This tells Terraform that the resource address `aws_instance.old_name` has moved to `module.web.aws_instance.web`. When you run `terraform init` or `terraform apply`, Terraform updates the state accordingly.

Hands-On Lab — Move a resource into a module

1. Create a simple config with `resource "local_file" "orphan" { ... }` and `terraform apply`.
2. Create a `module` directory and move the `local_file` block into the module, changing the address to `module.m.local_file.orphan`.
3. Add a `moved` block in the root module mapping the old address to the new module address.
4. Run `terraform init` then `terraform plan` and verify no recreation occurs.

---

## 7c.9: Secrets handling best practices and lab

Terraform (and providers) sometimes surface secrets in logs, state, or configuration. Follow these rules to reduce risk:

- Mark outputs and variables as `sensitive = true` when they contain secrets so `terraform output` and plan output do not print them.
- Avoid hard-coding secrets in `.tf` files or checked-in files. Use environment variables, secret managers (Vault, AWS Secrets Manager, Azure Key Vault), or CI/CD secret stores.
- Do not enable `TF_LOG=TRACE/DEBUG` in production for long periods; logs may contain credential material. If you must capture verbose logs for troubleshooting, rotate the credentials immediately afterward.
- Use remote backends with encryption-at-rest (S3 with SSE, GCS, Azure Blob Storage) and restrict access with IAM roles. For extra safety, enable server-side encryption with customer-managed keys.

Secrets Lab — Use a secret manager

1. Store a secret in Vault or your cloud provider's secret manager.
2. Configure your provider to read the secret at runtime (provider-specific example: `data "vault_generic_secret" ...`).
3. Reference the secret in resources/locals but mark outputs referencing it as `sensitive = true`.
4. Run `terraform plan` and verify secrets do not appear in plan output or `terraform show` unless explicitly requested with `-json` (which should be used cautiously).

### Logging and Secrets — Safety Guidance

- Avoid enabling `TF_LOG=TRACE/DEBUG` in environments where secrets (API keys, client secrets) might be emitted to logs. Provider requests and responses can include sensitive fields.
- In CI, prefer `TF_LOG=WARN` and capture logs to an artifact only for failed runs. Use your CI provider's log-redaction features to mask known secret patterns.
- If you must collect verbose logs for support, rotate any credentials used during the debug session and store logs in a restricted location.


---

## Objective 7: Quiz

> Good luck, Have Fun 😎 and don't forget to Debug your way through these series of questions below


**Question 1 (True/False):**

The `import` block can generate a complete Terraform configuration for the imported resource using the `-generate-config-out` flag during `terraform plan`.

**Question 2 (Multiple Choice):**

You have an existing EC2 instance `i-abc123` that you want to bring under Terraform management. You write a `resource "aws_instance" "managed"` block and an `import` block with `id = "i-abc123"` and `to = aws_instance.managed`. What command do you run to complete the import?
```
⬜ `terraform import`
⬜ `terraform apply`
⬜ `terraform plan -import`
⬜ `terraform state push`
```
**Question 3 (True/False):**

After a successful import, you can remove the `import` block from your configuration, and Terraform will continue to manage the resource normally.

**Question 4 (Multiple Choice):**

Which CLI command lists all resources currently tracked in the Terraform state file?
```
⬜ `terraform state show`
⬜ `terraform show`
⬜ `terraform state list`
⬜ `terraform state pull`
```
**Question 5 (Multiple Answer):**

Which of the following are valid `terraform state` subcommands? (Select **THREE**)
```
⬜ `terraform state list`
⬜ `terraform state read`
⬜ `terraform state mv`
⬜ `terraform state rm`
⬜ `terraform state refresh`
```
**Question 6 (True/False):**

`terraform state push` can overwrite the remote state file unconditionally, without any safety checks.

**Question 7 (Multiple Choice):**

What is the recommended `TF_LOG` level for capturing detailed information needed for a bug report?
```
⬜ `INFO`
⬜ `DEBUG`
⬜ `TRACE`
⬜ `WARN`
```
**Question 8 (Multiple Answer):**

Which of the following environment variables affect Terraform's logging behavior? (Select **TWO**)
```
⬜ `TF_LOG`
⬜ `TF_VAR_log_level`
⬜ `TF_LOG_PATH`
⬜ `TF_LOG_CONSOLE`
```
**Question 9 (True/False):**

The `terraform state show` command can display the attributes of a data source if it is present in the state file.

**Question 10 (Multiple Choice):**

You want to import multiple existing S3 buckets into a single Terraform configuration. What is the best approach with Terraform 1.5+?
```
⬜ Run `terraform import` once for each bucket.
⬜ Use one `import` block with `for_each` and a destination resource with `for_each`.
⬜ Manually add each bucket's ID to the state file using a text editor.
⬜ Use the `terraform state merge` command.
```
**Question 11 (True/False):**

Setting `TF_LOG_PROVIDER=DEBUG` enables debug logging for Terraform Core but not for any providers.

**Question 12 (Multiple Choice):**

What does the `terraform state rm` command do?
```
⬜ It destroys the resource in the cloud and removes it from state.
⬜ It removes the resource from the state file without destroying it.
⬜ It renames the resource in the state file.
⬜ It moves the resource to a different state file.
```
**Question 13 (True/False):**

The `import` block supports importing resources into child modules using the `to = module.MODULE_NAME.RESOURCE_TYPE.NAME` address format.

**Question 14 (Multiple Answer):**
Which of the following are limitations of the `import` block? (Select **TWO**)
```
⬜ It cannot import resources that were created by Terraform.
⬜ It does not automatically detect and add resource dependencies.
⬜ Not all provider resource types support import.
⬜ It requires the resource to be destroyed before import.
```
**Question 15 (True/False):**
`terraform show -json` provides a machine-readable output that is considered a stable interface and safe for production tooling.

<details>

<summary>Show Answers</summary>

1. True - The `import` block can generate a complete Terraform configuration for the imported resource using the `-generate-config-out` flag during `terraform plan`. This is a modern feature that helps you quickly create a configuration that matches the imported resource's current state.
2. `terraform apply` - After defining the `import` block, you run `terraform apply` to execute the import process. The `import` block is declarative and integrated into the plan/apply workflow, so you don't use the old `terraform import` command.
3. True - After a successful import, the `import` block has done its job of bringing the resource into state. You can remove the `import` block from your configuration, and Terraform will continue to manage the resource as normal.
4. `terraform state list` - This command lists all resource addresses currently tracked in the Terraform state file. It shows both managed resources and cached data sources.
5. `terraform state list`, `terraform state mv`, `terraform state rm` - These are valid `terraform state` subcommands. `terraform state read` is not a valid command; you use `terraform state show` to read resource attributes. `terraform state refresh` is also not a valid command; you use `terraform refresh` to update the state with real-world values.
6. False - `terraform state push` has safety checks that prevent unintentional overwrites. It checks the `lineage` and `serial` values to ensure you are not pushing an outdated state. You must use `-force` to bypass these checks, which is not recommended unless you are certain.
7. `TRACE` - The `TRACE` log level provides the most detailed information, which is recommended for bug reports to give developers the best chance of diagnosing the issue.
8. `TF_LOG`, `TF_LOG_PATH` - These environment variables control Terraform's logging behavior. `TF_LOG` sets the log level, while `TF_LOG_PATH` specifies a file to write logs to. `TF_VAR_log_level` is not a valid environment variable for logging, and `TF_LOG_CONSOLE` does not exist.
9. True - If a data source is present in the state file (e.g., because it was cached during a plan), `terraform state show` can display its attributes just like any other resource.
10. Use one `import` block with `for_each` and a destination resource with `for_each` - This is the most efficient way to import multiple resources in Terraform 1.5+. You can define a single `import` block that iterates over a map of identifiers and resource addresses, rather than running separate commands for each bucket.
11. False - Setting `TF_LOG_PROVIDER=DEBUG` enables debug logging for all providers, but it does not affect Terraform Core logging. To enable debug logging for Terraform Core, you would set `TF_LOG=DEBUG`.
12. It removes the resource from the state file without destroying it - The `terraform state rm` command removes the specified resource from the state file, but it does not destroy the actual resource in the cloud. This can be useful for "orphaning" a resource that you want to manage outside of Terraform.
13. True - The `import` block supports importing resources into child modules using the appropriate resource address format. You can specify the destination as `to = module.MODULE_NAME.RESOURCE_TYPE.NAME` to import directly into a module's resource block.
14. It does not automatically detect and add resource dependencies, Not all provider resource types support import - The `import` block does not analyze the resource's relationships or dependencies, so you must manually add any necessary `depends_on` or other references in your configuration. Additionally, not all resource types in every provider support import, so you should check the provider documentation for limitations.
15. False - The `terraform show -json` output is not considered a stable interface. The study guide explicitly warns that the JSON format may change between Terraform versions, so it should only be used for tooling that you maintain and update alongside Terraform. It is not guaranteed to be stable for long-term production use.

</details>


---

You have come a long way from what is Terraform to how to maintain infrastructure with Terraform. In the next and final objective, we will explore HCP Terraform, HashiCorp's managed Terraform service that provides additional features for collaboration, governance, and automation.

Let's proceed to [**Objective 8: Use HCP Terraform**](./08-HCP-Terraform.md), the final objective of the Associate certification.