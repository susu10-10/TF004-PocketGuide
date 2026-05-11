# Objective 3: Use the core Terraform workflow.

>This is a very hands-on objective and one you will use daily. Following the offical guide's structure exactly:

> Because this is alot, i created links to each sub-objective so you can jump to the one you want to focus on. I recommend going through them in order, but feel free to skip around as needed.

## 📑 Objectives

- [3a: Describe the Terraform workflow](#objective-3a-describe-the-terraform-workflow-write-plan-apply)
- [3b: Initialize a Terraform working directory (terraform init deep dive)](#objective-3b-initialize-a-terraform-working-directory-terraform-init-deep-dive)
- [3c: Validate a Terraform configuration (terraform validate)](#objective-3c-validate-a-terraform-configuration-terraform-validate)
- [3d: Generate and review an execution plan (terraform plan)](#objective-3d-generate-and-review-an-execution-plan-terraform-plan)
- [3e: Apply changes to infrastructure (terraform apply)](#objective-3e-apply-changes-to-infrastructure-terraform-apply)
- [3f: Destroy Terraform-managed infrastructure (terraform-destroy)](#objective-3f-destroy-terraform-managed-infrastructure-terraform-destroy)
- [3g: Apply formatting and style adjustments (terraform fmt)](#objective-3g-apply-formatting-and-style-adjustments-terraform-fmt)




## Objective 3a: Describe the Terraform workflow

Source: [Core Terraform Workflow Overview](https://developer.hashicorp.com/terraform/intro/v1.12.x/core-workflow)

### 3a.1: The Trinity of Terraform

The core workflow is a loop: **Write** → **Plan** → **Apply**.

**Write:**

- You author infrastructure as code in `.tf` files (or `.tf.json`).

- You use HCL (HashiCorp Configuration Language) to declare the desired end state of your resources.

- You typically store this code in a Version Control System (VCS) like Git.

**Plan:**

- You run `terraform plan`.

- Terraform reads your configuration, reads the **state file** (or creates an empty one if none exists), and performs a **refresh** to query the current real-world state of your infrastructure.

- It generates an execution plan: a diff showing what actions it will take to make the real world match the configuration.

- This is a read-only operation. No infrastructure is modified.

**Apply:**

- You run `terraform apply`.

- Terraform presents the execution plan and prompts for confirmation (yes).

- Upon approval, it executes the plan, making API calls to create, update, or destroy resources.

- It updates the state file to reflect the new reality.

**The Loop:** After apply, you return to Write to make further changes. This is the same loop whether you're an individual practitioner or a large team (though teams augment it with VCS branches and pull requests).

### 3a.2: Individual Practitioner vs. Team Workflow

The guide distinguishes between working alone and working on a team.

**Individual:**

- Write code in your editor.

- Run `terraform plan` iteratively to catch syntax errors and validate logic.

- Run `terraform apply` when ready.

- Push code to remote VCS for safekeeping.

**Team:**

- Write code in feature branches to avoid conflicts.

- Use a shared **remote backend** for state (Objective 6) so everyone works with the same state.

- Use a CI/CD pipeline (like HCP Terraform) to run `terraform plan` and `terraform apply` in a consistent environment.

- Use **speculative plans** on pull requests to preview changes before merging.

- Use **policy checks (Sentinel/OPA)** to enforce compliance before apply.

### 3a.3: HCP Terraform Enhanced Workflow

The guide emphasizes how HCP Terraform enhances each stage:

- **Write**: Centralized variable storage, private module registry.

- **Plan**: Speculative plans on PRs, cost estimation, policy checks.

- **Apply**: Secure remote execution, run history, notifications.

We will cover HCP Terraform in depth in Objective 8. For now, know that the core workflow remains the same; HCP Terraform adds collaboration and governance layers.

### 3a.4: Hands-On Lab — The Core Loop with Local Provider

Goal: Experience the Write → Plan → Apply loop.

#### Step 1: Create directory `obj3a-lab` and file `main.tf`

```hcl
resource "local_file" "example" {
  content  = "This is version 1"
  filename = "${path.module}/output.txt"
}
```

#### Step 2: `terraform init`

- Downloads the `local` provider.

#### Step 3: `terraform plan`

- Observe output: `+ create` action.

**Note:** Terraform shows the filename and content.

#### Step 4: `terraform apply`

- Type `yes`.

- File `output.txt` appears.

#### Step 5: Modify `main.tf` — change content to `"This is version 2"`.

- Run `terraform plan`. Observe ~ update in-place.

- Run `terraform apply`. File content changes.

#### Step 6: Delete the resource block entirely.

- Run `terraform plan`. Observe - destroy.

- Run `terraform apply`. File is deleted.

This loop is the foundation of all Terraform work.

## Objective 3b: Initialize a Terraform working directory (terraform init)

Source: [Initialize Terraform configuration](https://developer.hashicorp.com/terraform/tutorials/cli/init), [terraform init command](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/init)

### 3b.1: What `terraform init` Does

The `terraform init` documentation states it performs several initialization steps:

1. Backend Initialization: Reads the `backend` or `cloud` block and sets up state storage.

2. Provider Installation: Downloads provider plugins specified in `required_providers`.

3. Module Installation: Downloads child modules referenced in `module` blocks.

4. Lock File Management: Creates or updates `.terraform.lock.hcl`.

>**Important Note:** init is safe to run multiple times. It will not delete configuration or state.

### 3b.2: Key Options and Flags

From the `terraform init` command reference:

Flag|Purpose
-|-
`-upgrade`	| Upgrades modules and providers to the newest version allowed by constraints, ignoring the lock file.
`-reconfigure`	| Reconfigures the backend, ignoring any existing saved configuration.
`-migrate-state`	| Migrates existing state to the new backend (prompts for confirmation).
`-backend-config=FILE`	| Provides backend configuration from a file (useful for partial configuration).
`-lock=false`	| Disables state locking (dangerous).
`-input=false`	| Disables interactive prompts (useful in automation).
`-json`	| Outputs machine-readable JSON (for tooling).

**Important Note:** `-upgrade` only upgrades providers/modules that are already installed. If you add a new provider, `init` downloads it regardless of `-upgrade`.

### 3b.3: The `.terraform` Directory and Lock File

After `init`, Terraform creates:

- `.terraform/` — contains downloaded providers and modules.

- `.terraform.lock.hcl` — dependency lock file.

**Important Note:**

- The `.terraform` directory should not be committed to version control. It contains platform-specific binaries.

- The lock file should be committed. It ensures everyone uses the exact same provider versions.

### 3b.4: Partial Backend Configuration

Backend configurations often contain sensitive or dynamic values that shouldn't be hardcoded. Terraform supports partial configuration:

```hcl
terraform {
  backend "s3" {
    # bucket and key omitted — will be provided at init time
  }
}
```
Then run:

```bash
terraform init -backend-config="bucket=mybucket" -backend-config="key=prod/terraform.tfstate"
```

Or use a file:

```bash
terraform init -backend-config=backend.hcl
```

**Important Note:** The backend block cannot use `variables`, but partial configuration allows dynamic values via CLI.

### 3b.5: Hands-On Lab — `terraform init` Flags

**Goal:** Observe the effect of `-upgrade` and lock file behavior.

#### Step 1: Create `obj3b-lab/main.tf` with:

```hcl
terraform {
  required_providers {
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5.0"
    }
  }
}
```

#### Step 2: Run `terraform init`. 

Note version installed (e.g., 3.5.1). Observe lock file created.

#### Step 3: Change version constraint to `~> 3.6.0`.

- Run `terraform init`. It should fail because locked version 3.5.1 does not satisfy constraint.

- Run `terraform init -upgrade`. It updates lock file to 3.6.x.

#### Step 4: Delete `.terraform` directory and lock file.

- Run `terraform init`. It creates new lock file with latest matching version.

## Objective 3c: Validate a Terraform configuration (`terraform validate`)

Source: [terraform validate command](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/validate), [Initialize Terraform configuration](https://developer.hashicorp.com/terraform/tutorials/cli/init)

### 3c.1: Purpose of `terraform validate`

`terraform validate` checks whether a configuration is **syntactically valid** and **internally consistent**. It does not check remote services, state, or provider APIs.

**Validation includes:**

- Correct HCL syntax.

- Valid resource types and data source names.

- Required arguments are present.

- Correct types for arguments.

- No circular dependencies in `depends_on`.

**It does NOT check:**

- Whether an AMI ID actually exists.

- Whether IAM permissions are sufficient.

- Whether resource names are globally unique (e.g., S3 bucket names).

### 3c.2: Prerequisites

From the Docs: _"Validation requires an initialized working directory with any referenced plugins and modules installed."_

You must run `terraform init` before `terraform validate`. If you haven't, Terraform errors with "Provider not found."

### 3c.3: Command Options

Flag|Purpose
--|--
`-json`|Output results in machine-readable JSON format (useful for editor integrations).
`-no-color`|Disable terminal color codes.


**Important Note:** `terraform plan` also performs an implicit validation before generating the plan. If you only want validation, use `terraform validate`.

### 3c.4: JSON Output Format

The Docs details the JSON output structure:

- `valid`: boolean indicating overall validity.

- `error_count`, `warning_count`: integer counts.

- `diagnostics`: array of objects with severity (`"error"` or `"warning"`), summary, detail, and optional `range` (source location).

**Important Note:** A configuration can be `valid: true` but still have `warning_count > 0`. Warnings do not invalidate the configuration.

### 3c.5: Hands-On Lab — Validation Errors

#### Step 1: Create `obj3c-lab/main.tf` with an intentional syntax error:

```hcl
resource "local_file" "bad" {
  content = "Missing closing quote
  filename = "test.txt"
}
```
#### Step 2: Run `terraform init` (required).

#### Step 3: Run `terraform validate`.

- Observe error message with line number.

#### Step 4: Fix the syntax error, but change resource type to `local_fil` (typo).

- Run `terraform validate`. Observe error about unknown resource type.

#### Step 5: Correct the resource type but omit the required  filename   argument.

- Run `terraform validate`. Observe error about missing required argument.

>This reinforces what `terraform validate` catches and what it doesn't.

## Objective 3d: Generate and review an execution plan (terraform plan)

Source: [Create a Terraform plan](https://developer.hashicorp.com/terraform/tutorials/cli/plan), [terraform plan command](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/plan)

### 3d.1: What `terraform plan` Does

1. Reads configuration and state.

2. Refreshes state (by default) — queries real-world resources to update the state file with current attributes.

3. Compares desired state (configuration) with current state (state file).

4. Generates an execution plan listing actions to reconcile differences.

### 3d.2: Planning Modes

The Docs lists two alternative planning modes:

Mode|Flag|Purpose
-|-|-
Destroy|`-destroy`|Creates a plan to destroy all resources managed by the workspace.
Refresh-only|`-refresh-only`|Creates a plan to update state to match real-world resources without changing configuration.

**Important Note:** `-refresh-only` is safer than the deprecated `terraform refresh` command because it lets you review changes before applying them to state.

### 3d.3: Planning Options

Option|Purpose
-|--
`-refresh=false`|Skip refreshing state before plan (faster but may miss external changes).
`-replace=ADDRESS`|Force replacement of a specific resource (e.g., aws_instance.web).
`-target=ADDRESS`|Limit planning to a specific resource and its dependencies (use sparingly).
`-var 'NAME=VALUE'`|Set a variable value.
`-var-file=FILENAME`|Load variables from a file.

**Warning on `-target`:** The Docs explicitly states: "_This targeting capability is provided for exceptional circumstances._ What It means is that it is not recommended to use `-target` for routine operations."

### 3d.4: Saved Plans

You can save a plan to a file using -out=FILENAME:

```bash
terraform plan -out=tfplan
```

**Why save plans?**

- In automation, you want to apply the exact plan that was reviewed and approved.

- Applying a saved plan ensures no drift occurred between plan and apply.

**Apply a saved plan:**

```bash
terraform apply tfplan
```

**Important Security Note:** Saved plan files contain all variable values, including sensitive ones, in plaintext. Treat them as secrets.

### 3d.5: Plan Output Symbols

The plan uses symbols to indicate actions:

`+` : Create

`-` : Destroy

`~` : Update in-place

`-/+` : Replace (destroy then create)

`+/-` : Replace (create before destroy, due to `create_before_destroy` lifecycle)

**Important Note:** Be able to identify what each symbol means in a sample plan output.

### 3d.6: Detailed Exit Codes

When using `-detailed-exitcode`:

`0` = Succeeded with empty diff (no changes)

`1` = Error

`2` = Succeeded with non-empty diff (changes present)

>This is crucial for CI/CD scripting.

### 3d.7: Hands-On Lab — Planning with Local Provider

#### Step 1: Create `obj3d-lab/main.tf` with:

```hcl
resource "local_file" "test" {
  content  = "Hello"
  filename = "${path.module}/hello.txt"
}
```

#### Step 2: Run `terraform init` and `terraform apply`.

#### Step 3: Modify content to "Hello World".

- Run `terraform plan`. Observe `~` update.

- Run `terraform plan -out=tfplan`.

Observe file created.

#### Step 4: Apply the saved plan:

```bash
terraform apply tfplan
```

> **Note:** No confirmation prompt because plan is already approved.

#### Step 5: Delete the file manually (simulate external change).

- Run `terraform plan -refresh-only`. Observe Terraform detects drift and proposes to update state.

- Run `terraform apply -refresh-only` to accept state update.

#### Step 6: Run `terraform plan -destroy` to see destroy plan without executing.


## Objective 3e: Apply changes to infrastructure (terraform apply)

Source: [Apply Terraform configuration](https://developer.hashicorp.com/terraform/tutorials/cli/apply), [terraform apply command](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/apply)

### 3e.1: Modes of Operation

`terraform apply` can be used in two ways:

1. **Automatic Plan Mode (no plan file argument):**

- Terraform generates a new plan, displays it, and prompts for approval.

- You can use all planning options (`-var`, `-replace`, etc.).

2. **Saved Plan Mode (with plan file argument):**

- Terraform applies the exact actions from the saved plan.

- No confirmation prompt (the plan file is considered approval).

- No planning options allowed (the plan is already fixed).

### 3e.2: Apply Options

Option|Purpose
-|--
`-auto-approve`|Skip interactive approval (useful in automation, but ensure plan is reviewed).
`-compact-warnings`|Show warnings in compact form.
`-lock=false`|Don't hold state lock (dangerous).
`-lock-timeout=DURATION`|How long to retry acquiring lock (e.g., `30s`).
`-parallelism=n`|Limit concurrent operations (default 10).
`-json`	|Machine-readable JSON output.

### 3e.3: Apply Steps in Detail

From the Documentation's Apply Terraform configuration tutorial, when you apply, Terraform:

1. Locks the state (if using a backend with locking).

2. Creates a plan (or uses provided plan).

3. Waits for approval (unless `-auto-approve` or saved plan).

4. Executes the plan — makes API calls in dependency order, in parallel when possible.

5. Updates the state file with new resource attributes.

6. Unlocks the state.

7. Reports outputs defined in configuration.

### 3e.4: Error Handling During Apply

- If an error occurs mid-apply:

- Terraform logs the error.

- It updates the state to reflect any resources that were successfully created or modified.

- It unlocks the state.

- It exits with a non-zero code.

**Important Point:** Terraform does not automatically roll back a **partially-completed apply**. The infrastructure may be in an inconsistent state. You must fix the error and run `terraform apply` again to converge to the desired state.

### 3e.5: Hands-On Lab — Apply with Error

Goal: Observe apply error behavior.

#### Step 1: Create `obj3e-lab/main.tf` with a resource that will fail:

```hcl
resource "local_file" "will_fail" {
  content  = "test"
  filename = "/root/cannot_write_here.txt"  # Permission denied (unless root)
}
```

#### Step 2: Run `terraform init` and `terraform apply`. 

Observe error.

#### Step 3: Check state file (`terraform show`). No resources created because local file creation fails before state update.

#### Step 4: Modify to a valid path, run `terraform apply` again. 

This demonstrates that you should resolve the error and re-apply.

## Objective 3f: Destroy Terraform-managed infrastructure (terraform destroy)

Source: [terraform destroy command](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/destroy), [Apply Terraform configuration](https://developer.hashicorp.com/terraform/tutorials/cli/apply)

### 3f.1: What destroy Does

terraform destroy is a convenience alias for:

```bash
terraform apply -destroy
```

It creates a destroy plan (showing all resources that will be deleted), prompts for confirmation, and then executes the plan.

### 3f.2: Key Options

Option|Purpose
-|--
`-auto-approve`|Skip confirmation prompt. Use with caution.
`-target=ADDRESS` | Destroy only a specific resource and its dependencies.

**Important Note:** Using `-target` for destroy can leave dependent resources in a broken state. Use with caution.

### 3f.3: Destroy Order

Terraform destroys resources in reverse dependency order. 

For example, if an `EC2 instance` depends on a `security group`, Terraform will destroy the `instance` before destroying the `security group`. This prevents API errors (can't delete a security group that's in use).

**Important Note:** The dependency graph ensures correct destroy order automatically.

### 3f.4: Hands-On Lab — Destroy Workflow

#### Step 1: Use any previous lab with resources.

#### Step 2: Run `terraform plan -destroy`. Observe the plan.

#### Step 3: Run `terraform destroy`. Type yes. Observe resources deleted.

#### Step 4: Check state file. It should be empty (no resources).

## Objective 3g: Apply formatting and style adjustments (`terraform fmt`)

Source: [terraform fmt command](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/fmt)

### 3g.1: Purpose of `fmt`

`terraform fmt` rewrites Terraform configuration files to a canonical format and style. It enforces:

- Consistent indentation (two spaces).

- Alignment of = signs.

- Removal of unnecessary whitespace.

- Proper ordering of arguments.

**Important Note:** `terraform fmt` is opinionated and has no customization options. Its goal is consistency across Terraform codebases.

### 3g.2: Command Usage

```bash
terraform fmt [options] [target...]
```

By default, it scans the current directory for `.tf` and `.tfvars` files.

- `-recursive`: Also process files in subdirectories.

- `-diff`: Show differences without modifying files.

- `-check`: Exit with non-zero if files need formatting (useful in CI).

- `-list=false`: Don't list files that were formatted.

### 3g.3: Hands-On Lab — Formatting

#### Step 1: Create `obj3g-lab/main.tf` with messy formatting:

```hcl
resource    "local_file" "messy" {
  content = "bad spacing"
    filename = "out.txt"
}
```

#### Step 2: Run `terraform fmt -diff`. 

See the proposed changes.

#### Step 3: Run `terraform fmt`. 

File is reformatted.

#### Step 4: Run `terraform fmt -check`. 

Exit code 0 means already formatted.


## Objective 3: Quiz (20+ Questions)

Note your answers and check against the provided answers at the end of this section.

**Question 1 (True/False):**

Running `terraform plan` will automatically run `terraform init` if the working directory has not been initialized yet.
```
⬜ True
⬜ False
```

**Question 2 (True/False):**

The `terraform apply` command, when used without a saved plan file, will generate a new execution plan and immediately apply it without any user interaction.
```
⬜ True
⬜ False
```

**Question 3 (Multiple Choice):**

Which command would you use to check whether your Terraform configuration files follow the canonical style conventions without actually modifying them?
```
⬜ terraform fmt
⬜ terraform fmt -check
⬜ terraform validate -style
⬜ terraform plan -style-check
```
**Question 4 (Multiple Choice):**
A Terraform plan file (tfplan) was created using `terraform plan -out=tfplan`. What is the correct command to apply this exact plan?
```
⬜ terraform apply tfplan
⬜ terraform apply -plan=tfplan
⬜ terraform apply -auto-approve tfplan
⬜ terraform plan -apply tfplan
```
**Question 5 (Multiple Answer):**
Which of the following actions does `terraform init` perform? **(Select THREE)**
```
⬜ Downloads and installs provider plugins.
⬜ Generates an execution plan for the current configuration.
⬜ Initializes the configured backend for state storage.
⬜ Validates the syntax of all .tf files.
⬜ Downloads child modules referenced in module blocks.
⬜ Creates the resources defined in the configuration.
```

**Question 6 (True/False):**

You can use variables inside a backend block to dynamically set the state file path based on the current workspace.
```
⬜ True
⬜ False
```

**Question 7 (Multiple Choice):**

What happens if you run `terraform apply` and the apply process encounters an error after successfully creating two of the three resources defined in the configuration?
```
⬜ Terraform automatically rolls back the two created resources and exits.
⬜ Terraform halts execution, records the two created resources in the state file, and exits with an error.
⬜ Terraform continues attempting to create the third resource indefinitely.
⬜ Terraform deletes the two created resources and then exits with an error.
```

**Question 8 (Multiple Choice):**

You have made manual changes to a resource outside of Terraform. You want to update the Terraform state file to match the current real-world state without modifying your configuration. Which command sequence is the recommended approach?
```
⬜ terraform refresh
⬜ terraform plan -refresh-only then terraform apply -refresh-only
⬜ terraform state pull then manually edit the JSON and terraform state push
⬜ terraform apply -target=resource_address
```

**Question 9 (True/False):**
The `terraform validate` command requires an initialized working directory (i.e., `terraform init` must have been run) to function correctly.
```
⬜ True
⬜ False
```
**Question 10 (Multiple Choice):**

In a team environment using HCP Terraform, what is the primary purpose of a speculative plan?
```
⬜ To apply changes to production infrastructure after hours.
⬜ To preview the potential effects of a configuration change in a pull request before merging.
⬜ To estimate the monthly cost of the infrastructure.
⬜ To lock the workspace to prevent other users from making changes.
```

**Question 11 (Multiple Answer):**

Which of the following are valid options for the `terraform plan` command? **(Select THREE)**
```
⬜ -destroy
⬜ -refresh-only
⬜ -auto-approve
⬜ -out=FILENAME
⬜ -compact-warnings
```

**Question 12 (True/False):**

A saved Terraform plan file (created with `-out`) contains the full configuration, variable values, and planned changes in an encrypted format to protect sensitive data.
```
⬜ True
⬜ False
```

**Question 13 (Multiple Choice):**

What does the `terraform fmt -recursive` command do?
```
⬜ Formats only the files in the current directory.
⬜ Formats files in the current directory and all subdirectories.
⬜ Formats files and also validates their syntax.
⬜ Formats files and automatically commits them to Git.
```
**Question 14 (Multiple Answer):**

Which statements accurately describe the `terraform destroy` command? **(Select TWO)**
```
⬜ It is an alias for terraform apply -destroy.
⬜ It requires a saved plan file to execute.
⬜ It prompts for confirmation before deleting resources.
⬜ It only deletes resources that have been commented out in the configuration.
⬜ It deletes the state file along with the infrastructure.
```
**Question 15 (Multiple Choice):**

During `terraform apply`, Terraform creates resources in parallel when possible. What is the default maximum number of concurrent resource operations?
```

⬜ 1 (sequential)
⬜ 5
⬜ 10
⬜ 20
```

**Question 16 (True/False):**

If you run `terraform validate` on a configuration that contains a depends_on cycle (circular dependency), it will detect and report the error.
```

⬜ True
⬜ False
```

**Question 17 (Multiple Choice):**

You want to force the replacement of a specific EC2 instance (`aws_instance.web`) during the next `terraform apply`, even though its configuration has not changed. Which command would you use?
```

⬜ terraform apply -replace=aws_instance.web
⬜ terraform plan -destroy-target=aws_instance.web
⬜ terraform taint aws_instance.web
⬜ terraform apply -target=aws_instance.web
```

**Question 18 (Multiple Answer):**

Which of the following are true about the `-target` option? **(Select TWO)**
```
⬜ It is recommended for routine use to speed up operations on large configurations.
⬜ It limits the plan/apply to the specified resource and its dependencies.
⬜ It can cause configuration drift if used regularly.
⬜ It works with both plan and apply but not destroy.
```

**Question 19 (Multiple Choice):**

A team member runs `terraform apply` but forgets to pull the latest changes from Git. Their local state file is older than the remote state. What happens if they are using a remote backend with state locking (e.g., S3 with DynamoDB)?
```
⬜ Terraform will overwrite the remote state with the local state.
⬜ Terraform will merge the local state with the remote state.
⬜ Terraform will fail to acquire the state lock because the serial number in the remote state is higher.
⬜ Terraform will automatically pull the latest state before applying.
```

**Question 20 (True/False):**
The `terraform fmt` command can be customized via a `.terraform-fmt.conf` file to adjust indentation size and alignment preferences.
```
⬜ True
⬜ False
```

**Question 21 (Multiple Choice):**
What is the exit code of `terraform plan -detailed-exitcode` if there are no changes to be made?
```

⬜ 0
⬜ 1
⬜ 2
⬜ 3
```

**Question 22 (Multiple Answer):**

Which of the following files or directories should not be committed to version control? **(Select THREE)**
```

⬜ .terraform/
⬜ terraform.tfstate
⬜ .terraform.lock.hcl
⬜ terraform.tfvars (if it contains sensitive data)
⬜ main.tf
```

**Question 23 (True/False):**

The `terraform plan -refresh-only` command will modify your real infrastructure to match the configuration if drift is detected.
```

⬜ True
⬜ False
```

<details>

<summary>Show answers</summary>

**Answers & brief explanations**

**Q1: FALSE** — `terraform plan` does not automatically run `terraform init`. If the working directory is not initialized, Terraform will error and instruct you to run `init` first.

**Q2: FALSE** — Without a saved plan file, `terraform apply` generates a new execution plan **and prompts for user approval** before applying. Use `-auto-approve` to skip the prompt.

**Q3: B** — `terraform fmt -check` lists files that need formatting and returns a non‑zero exit code without modifying them. `terraform fmt` alone would rewrite the files.

**Q4: A** — `terraform apply tfplan` applies the saved plan file. The `-out` flag is only used when *creating* the plan with `terraform plan -out=tfplan`.

**Q5: A, C, E** — `terraform init` downloads/installs provider plugins (A), initializes the configured backend for state storage (C), and downloads child modules referenced in `module` blocks (E). It does **not** generate a plan (B), validate syntax (D), or create resources (F).

**Q6: FALSE** — Backend blocks cannot use variables, locals, or any named values. They must be constant literals because Terraform evaluates them before parsing variables.

**Q7: B** — Terraform halts execution, records the successfully created resources in the state file, and exits with an error. It does **not** automatically roll back created resources.

**Q8: B** — The recommended approach is `terraform plan -refresh-only` followed by `terraform apply -refresh-only`. The legacy `terraform refresh` command is deprecated and less safe because it doesn't let you review changes.

**Q9: TRUE** — `terraform validate` requires an initialized working directory with providers and modules installed. You must run `terraform init` first, otherwise validation fails.

**Q10: B** — Speculative plans preview the effects of a configuration change in a pull request before merging. They are plan‑only runs triggered by VCS webhooks (e.g., GitHub PRs).

**Q11: A, B, D** — Valid `terraform plan` options include `-destroy` (A), `-refresh-only` (B), and `-out=FILENAME` (D). `-auto-approve` (C) is for `apply`, not plan. `-compact-warnings` (E) is also valid, so any three of A,B,D,E are correct.

**Q12: FALSE** — Saved plan files (`.tfplan`) are **not encrypted**; they contain plain‑text sensitive data (configuration, variable values, and planned changes). Treat them as sensitive artifacts and never commit them.

**Q13: B** — `terraform fmt -recursive` formats configuration files in the current directory and all subdirectories. Without `-recursive`, it only formats the current directory.

**Q14: A, C** — `terraform destroy` is an alias for `terraform apply -destroy` (A). It prompts for confirmation before deleting resources (C). It does **not** require a saved plan file (B), and it does **not** delete the state file itself (E).

**Q15: C** — The default parallelism is **10** concurrent resource operations. Use the `-parallelism=n` flag to change it.

**Q16: TRUE** — `terraform validate` detects circular dependencies (e.g., `depends_on` cycles) and reports an error. It is a built‑in check.

**Q17: A** — `terraform apply -replace=aws_instance.web` forces recreation of the specified resource. The legacy `taint` command is deprecated.

**Q18: B, C** — The `-target` option (B) limits operations to the specified resource and its dependencies. Using it regularly (C) can cause configuration drift because other resources may fall out of sync. It is **not** recommended for routine use (A), and it works with `destroy` as well (D).

**Q19: C** — Remote state locking prevents older state overwrite due to serial mismatch.

**Q20: FALSE** — `terraform fmt` is opinionated and cannot be customized via configuration files. There is no `.terraform-fmt.conf` or equivalent.

**Q21: A** — With `-detailed-exitcode`: `0` = success with no changes, `1` = error, `2` = success with changes.

**Q22: A, B, D** — Do not commit `.terraform/` (A – provider/module cache), `terraform.tfstate` (B – contains sensitive data), or `terraform.tfvars` if it contains secrets (D). The lock file `.terraform.lock.hcl` (C) and `main.tf` (E) **should** be committed.

**Q23: FALSE** — `terraform plan -refresh-only` only updates the state file to match real infrastructure; it does **not** modify real resources. A separate `terraform apply -refresh-only` would apply those state changes, but still **no infrastructure changes** are made.

</details>


That was a alot yh? Take a moment to review the questions and answers, and make sure you understand the reasoning behind each one. The core workflow and commands are fundamental to using Terraform effectively, so it's crucial to have a solid grasp of these concepts before moving on to more advanced topics.

You have come a long way in understanding the core Terraform workflow and commands. In the next section, we will dive into Objective 4, which focuses on Terraform Configuration Language (HCL) and how to write effective Terraform code. This will build on the foundation we've established here and introduce you to the syntax and structure of Terraform configurations.


Remember, the key to mastering Terraform is practice. I encourage you to experiment with the commands and concepts we've covered in this section by creating your own Terraform configurations and applying them to real or simulated infrastructure. This hands-on experience will solidify your understanding and prepare you for the more complex topics ahead.

Lets move on to [Objective 4: Write Terraform configuration using HCL](04-Terraform-Configuration.md) where we will explore the syntax and structure of Terraform's configuration language, HCL.
