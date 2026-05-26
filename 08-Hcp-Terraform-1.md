# Objective 8: Use HCP Terraform

We arrive at the final objective: **Objective 8: Use HCP Terraform**. This is where your Terraform skills are amplified by a platform that solves collaboration, governance, and operational challenges. HCP Terraform (formerly Terraform Cloud) is HashiCorp's SaaS for Terraform. Its features are directly tested on the Associate exam.

## Objectives

- [**Objective 8a: Use HCP Terraform to create infrastructure**](#objective-8a-use-hcp-terraform-to-create-infrastructure)

- [**Objective 8b: Describe HCP Terraform collaboration and governance features**](#objective-8b-describe-hcp-terraform-collaboration-and-governance-features)

- [**Objective 8c: Describe how to organize and use HCP Terraform workspaces and projects**](#objective-8c-describe-how-to-organize-and-use-hcp-terraform-workspaces-and-projects)

- [**Objective 8d: Configure and use HCP Terraform integration**](#objective-8d-configure-and-use-hcp-terraform-integration)

- [**Objective 8 Quiz**](#objective-8-quiz)


## Objective 8a: Use HCP Terraform to create infrastructure

*Source: [`Workspaces`](https://developer.hashicorp.com/terraform/cloud-docs/workspaces), [`HCP Terraform Workflow`](https://developer.hashicorp.com/terraform/cloud-docs/overview#terraform-workflow),[`HCP Terraform Overview`](https://developer.hashicorp.com/terraform/cloud-docs),[`Remote Operations`](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations)	[`HCP Terraform Get Started Collection`](https://developer.hashicorp.com/terraform/tutorials/cloud-get-started)*

### 8a.1: What is HCP Terraform?

HCP Terraform is a managed application that helps teams use Terraform together. Instead of running Terraform on your laptop and storing state in a shared S3 bucket (which you have to set up and manage yourself), HCP Terraform provides:

- **Remote state storage** – encrypted at rest, automatically versioned.
- **Remote execution** – Terraform `plan` and `apply` run on HCP Terraform's own servers, not your local machine.
- **Team collaboration** – role-based access control, policy enforcement, notifications.
- **Private module registry** – share and version modules within your organization.
- **Cost estimation** – see the cost impact of a plan before applying.
- **Policy as Code** – enforce compliance with Sentinel or OPA.
- **Drift detection and continuous validation** – detect infrastructure drift automatically.

### 8a.2: The Core Workflow in HCP Terraform

The guide's `Remote operations` section describes how runs work:

A **run** is a single `plan` and optional `apply` operation in a workspace. Runs can be initiated by:
- **VCS webhooks** – push to a branch triggers a run.
- **UI** – click "Start new run".
- **API** – programmatic integration.
- **CLI integration** – `terraform plan` and `terraform apply` from your local terminal, remotely executed.

**Run stages:**
1. **Pending** – queued, waiting for other runs to finish.
2. **Planning** – Terraform generates a plan.
3. **Policy Check** – Sentinel/OPA policies are evaluated against the plan.
4. **Awaiting Approval** – user reviews and approves the plan.
5. **Applying** – Terraform executes the changes.
6. **Completed** – resources provisioned, state updated.

### 8a.3: Remote Operations vs Local Execution

A workspace can be configured with either execution mode:

- **Remote** – `plan` and `apply` run on HCP Terraform's infrastructure. This enables all platform features: policy enforcement, cost estimation, notifications, secure variable storage.
- **Local** – HCP Terraform acts only as a state backend, similar to S3. All Terraform commands run on your local machine. Some features are not available (policy enforcement, cost estimation, run history).

You choose the execution mode when creating a workspace.

### 8a.4: Connecting a Configuration to HCP Terraform

To use HCP Terraform from your CLI, you add a `cloud` block to your `terraform` configuration:

```hcl
# The terraform block configures Terraform itself.
terraform {
  # The cloud block replaces the backend block. It connects your configuration
  # directly to an HCP Terraform organization and workspace.
  cloud {
    # 'organization' identifies the HCP Terraform organization to use.
    # This organization must already exist in HCP Terraform.
    organization = "my-company"

    # 'workspaces' specifies which workspace(s) this configuration belongs to.
    # You can use either a single name or a set of tags.
    workspaces {
      # 'name' is the exact workspace name. The workspace will be created
      # automatically if it doesn't already exist during terraform init.
      name = "my-app-prod"
    }
  }
}
```

**Explanation:**

- `terraform {` – Opens the top-level configuration block for Terraform itself.
- `cloud {` – Begins the HCP Terraform connection configuration. This is mutually exclusive with a `backend` block; you cannot have both.
- `organization = "my-company"` – The name of your HCP Terraform organization. This is the top-level grouping for your workspaces, teams, and settings.
- `workspaces {` – Defines which workspace(s) this configuration maps to.
- `name = "my-app-prod"` – Uses a single workspace with this exact name. When you run `terraform init`, Terraform will create this workspace if it doesn't exist.

**Alternative: Using tags instead of a single name:**

```hcl
terraform {
  cloud {
    organization = "my-company"
    workspaces {
      # 'tags' is a list of strings. The configuration will be associated
      # with ALL workspaces that have ALL of these tags.
      tags = ["app:web", "env:production"]
    }
  }
}
```

**Explanation:**

- `tags = ["app:web", "env:production"]` – When you use tags, the same configuration can apply to multiple workspaces. HCP Terraform finds workspaces that have all the listed tags. This is useful when you want to reuse the same configuration across environments (dev, staging, prod) but with different variable values per workspace.

---

### 8a.5: Authenticating with HCP Terraform

Before you can use the `cloud` block, you must authenticate the CLI to HCP Terraform.

```bash
# This command opens a browser for you to log in and generates an API token.
terraform login
```

**What `terraform login` does:**

1. Opens your web browser to `app.terraform.io`.
2. You log in (or confirm you're already logged in).
3. HCP Terraform generates an API token.
4. Terraform CLI stores this token in `~/.terraform.d/credentials.tfrc.json`.

**Alternative for automation:**
Set the `TFE_TOKEN` environment variable (or `TERRAFORM_CLOUD_TOKEN`).

```bash
export TFE_TOKEN="your-api-token-here"
```

### 8a.6: CLI-Driven Run Workflow

With the `cloud` block configured and authentication set up, you can use the standard Terraform CLI workflow. Commands like `terraform plan` and `terraform apply` are **remotely executed**.

```bash
terraform init        # Creates workspace if needed, sets up remote state
terraform plan        # Generates a speculative plan remotely, but does NOT queue a run
terraform apply       # Queues a run in the workspace, plan is generated and applied remotely
```

**Key behavior differences from local Terraform:**

- `terraform plan` produces a **speculative plan**—it shows what would happen but does not lock the workspace or queue a run. No approval is required.
- `terraform apply` queues a run in HCP Terraform, streams the plan output to your terminal, and waits for you to confirm. The actual `apply` executes on HCP Terraform's servers.
- You can also use saved plans: `terraform plan -out=tfplan` saves the plan locally, then `terraform apply tfplan` applies that exact plan remotely.

---

### 8a.7: Hands-On Lab — Conceptual CLI-Driven Workflow

Since HCP Terraform requires an account, we'll walk through the conceptual steps. (If you have a free HCP Terraform account, you can follow along.)

**Step 1: Create or use an existing HCP Terraform organization.**
- Sign up at `app.terraform.io`.
- Create an organization (e.g., `my-associate-lab`).

**Step 2: Configure your Terraform project.**

```hcl
# main.tf
terraform {
  cloud {
    organization = "my-associate-lab"
    workspaces {
      name = "obj8a-workspace"
    }
  }
}

# A simple resource that creates a random string—no cloud credentials needed.
resource "random_pet" "demo" {
  length = 2
}
```

**Step 3: Run the workflow.**
```bash
terraform login                                      # Authenticate to HCP Terraform
terraform init                                       # Creates obj8a-workspace in your org
terraform plan                                       # Remote speculative plan
terraform apply                                      # Queues and applies a run remotely
```

**Step 4: Observe in the HCP Terraform UI.**
- Navigate to your workspace `obj8a-workspace`.
- View the Runs tab—you'll see the completed run.
- View the States tab—the state file is stored there, encrypted.

---

### 8a.8: Mini-Quiz - Using HCP Terraform to Create Infrastructure

1. True/False: When using HCP Terraform with the CLI-driven workflow, `terraform apply` runs locally on your machine by default.
2. Multiple Choice: Which block in `terraform {}` is used to connect a Terraform configuration to HCP Terraform?
   A. `backend "remote"`
   B. `cloud`
   C. `cloud_backend`
   D. `tfe`
3. Multiple Answer: Which of the following can initiate a run in an HCP Terraform workspace? (Select THREE)
   A. Pushing code to a linked VCS repository.
   B. Running `terraform plan` locally.
   C. Clicking "Start new run" in the UI.
   D. Running `terraform apply` locally with the cloud block configured.
   E. Using the HCP Terraform API.

---

<details>

<summary>Show Answers</summary>

1. False. With the `cloud` block configured, `terraform apply` queues a run that executes remotely on HCP Terraform's infrastructure.
2. B. The `cloud` block is used to connect to HCP Terraform. The `backend "remote"` block is the older syntax for remote state only, not the full HCP Terraform integration.
3. A, C, D, E. All of these can initiate a run. `terraform plan` does not initiate a run; it generates a speculative plan locally without queuing anything.

</details>

## Objective 8b: Describe HCP Terraform collaboration and governance features

*Source: [`Explorer`](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/explorer),[`HCP Private Registry Overview`](https://developer.hashicorp.com/terraform/cloud-docs/registry),[`Change Requests Overview`](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/change-requests),[`HCP Terraform Policy Enforcement`](https://developer.hashicorp.com/terraform/cloud-docs/policy-enforcement),[`Projects Overview`](https://developer.hashicorp.com/terraform/cloud-docs/projects),[`Health`](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/health),[`HCP Terraform Teams`](https://developer.hashicorp.com/terraform/cloud-docs/users-teams-organizations/teams),[`Authenticate Providers with Dynamic Credentials`](https://developer.hashicorp.com/terraform/tutorials/cloud/dynamic-credentials),[`Drift Detection`](https://developer.hashicorp.com/terraform/tutorials/cloud/drift-detection),[`Variable Sets`](https://developer.hashicorp.com/terraform/tutorials/cloud/cloud-multiple-variable-sets),[`Detect Drift and Enforce Policies`](https://developer.hashicorp.com/terraform/tutorials/cloud/drift-and-policy),[`Enforce OPA Policies`](https://developer.hashicorp.com/terraform/tutorials/cloud/validation-enforcement)*

### 8b.1: Teams and Role-Based Access Control (RBAC)

HCP Terraform allows you to create teams of users and assign them permissions at the organization, project, and workspace levels.

**Team permissions are additive**: A user gets the highest level of access across all teams they belong to.

**Organization-level permissions:**

- `Manage all workspaces` – Full control over all workspaces.
- `Manage all projects` – Full control over projects.
- `View all workspaces` – Read-only access to workspace data.

**Project-level permissions:**

- `Admin` – Manage the project and all workspaces in it.
- `Maintain` – Create and manage workspaces within the project, but not the project itself.
- `Write` – Provision infrastructure (queue plans, apply) but not manage workspace settings.
- `Read` – View workspace data only.

**Workspace-specific permissions:**

- `Admin` – Full control over that workspace.
- `Read and write` – Manage variables, queue plans, apply.
- `Read` – View only.

### 8b.2: Policy as Code — Sentinel and OPA

HCP Terraform can enforce rules before a Terraform plan is applied. You define policies that check for security, compliance, or cost guardrails.

**Sentinel** is HashiCorp's policy language. **OPA** (Open Policy Agent) is a CNCF standard. You choose one per policy set.

**Policy enforcement levels:**

| Level | Behavior |
|-------|----------|
| `hard-mandatory` | The plan cannot be applied if the policy fails. The run stops. |
| `soft-mandatory` | The plan cannot be applied unless an authorized user overrides the failure. |
| `advisory` | The failure is shown as a warning but the run can proceed. |

**Policy sets** are collections of policies. They can be sourced from a VCS repository or uploaded via API. They can be scoped to specific workspaces, projects, or the entire organization.

### 8b.3: Change Requests

Change requests allow you to record action items directly on a workspace, such as "update deprecated module version" or "review security group rules." This creates a backlog of tasks tied to infrastructure.

### 8b.4: Notifications

Workspaces can send notifications (via Slack, email, webhooks) for run events: run started, run completed, run errored, policy failures, drift detected.

### 8b.5: Explorer for Workspace Visibility

The Explorer provides a queryable interface across your organization's workspaces, modules, providers, and Terraform versions. You can answer questions like "Which workspaces still use Terraform 1.0?" or "Which workspaces have drift?"

### 8b.6: Private Module Registry (publishing modules)

HCP Terraform's private registry lets your org publish modules for reuse. High-level steps:

1. Create a VCS repository containing your module (use `modules/my-module` layout and include `README.md`).
2. Tag releases with semantic versions (e.g., `v1.0.0`).
3. In the HCP Terraform UI, go to Registry → Publish Module and connect the VCS repo.
4. Add a `versions` constraint in your consuming configuration's `required_providers`/module source or use the registry source `app.terraform.io/my-org/my-module/aws`.

#### Best practices:

- Use semantic versioning and changelogs.
- Keep modules small and focused (one responsibility).
- Include examples and input/output documentation.
- Use a CI job to run `terraform validate` and `tflint` against module branches before publishing.

---

### 8b.7: Mini-Quiz - HCP Terraform Collaboration and Governance

1. True/False: In HCP Terraform, a user can be a member of multiple teams, and their effective permissions are the union of all team permissions.
2. Multiple Choice: Which policy enforcement level prevents a run from being applied unless an override is provided by an authorized user?
   A. `advisory`
   B. `hard-mandatory`
   C. `soft-mandatory`
   D. `warning`
3. Multiple Answer: Which of the following are collaboration features provided by HCP Terraform? (Select FOUR)
   A. Teams and role-based access control
   B. Policy enforcement with Sentinel/OPA
   C. Automatic application source code generation
   D. Change requests on workspaces
   E. Private module registry

---

<details>

<summary>Show Answers</summary>

1. True — Users may belong to multiple teams and receive the union of those team permissions.

2. C (`soft-mandatory`) — `soft-mandatory` requires an authorized override to allow an apply when the policy fails.

3. A, B, D, E — Teams/RBAC, policy enforcement, change requests, and a private module registry are collaboration features. (C is incorrect.)

</details>


## Objective 8c: Describe how to organize and use HCP Terraform workspaces and projects

*Source: [`Run Triggers`](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/run-triggers),[`Variable Sets`](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/variables/managing-variables#variable-sets),[`Projects Overview`](https://developer.hashicorp.com/terraform/cloud-docs/projects)	[`Connect Workspaces with Run Triggers`](https://developer.hashicorp.com/terraform/tutorials/cloud/cloud-run-triggers),[`Organize Workspaces with Projects`](https://developer.hashicorp.com/terraform/tutorials/cloud/projects)*

### 8c.1: Workspaces

A workspace is the primary unit of organization. It contains everything needed to manage a collection of infrastructure:

- Configuration version (the `.tf` files).
- State file (encrypted, versioned).
- Variables (Terraform and environment).
- Run history.
- Settings (execution mode, auto-apply, notifications).

**HCP Terraform workspaces vs. Terraform CLI workspaces:**

- **CLI workspaces** are alternate state files within the same directory, managed with `terraform workspace select`. They're for the same configuration applied to multiple environments (dev/staging/prod).
- **HCP Terraform workspaces** are fully independent. They can have entirely different configurations, variables, and execution modes. They're analogous to separate working directories in the CLI world.

**Workspace naming conventions:** Use structured names like `networking-prod-us-east` or `app-feature-a` to identify the component, environment, and region.

### 8c.2: Projects

Projects are containers for workspaces. They let you group related workspaces and assign permissions at the project level.

**Default Project:** Every organization has one. All workspaces belong to a project.

**Organizing with projects:** You might create projects for different business units (`platform`, `application`) or environments (`production`, `development`). Teams can be granted permissions on an entire project, which cascades to all workspaces within it.

**Project-scoped variable sets:** You can apply variable sets to a project so all current and future workspaces inherit them automatically.

### 8c.3: Variable Sets

Variable sets are reusable collections of variables that you can apply to multiple workspaces. They can be scoped globally (all workspaces in the organization), to specific projects, or to specific workspaces.

**Precedence (highest to lowest):**

1. Priority variable set
2. Workspace-specific variable
3. Variable set applied to the workspace (lexical order by name if multiple sets)
4. Environment variable `TF_VAR_*`
5. Default value in the `variable` block

**Note:** Priority variable sets (introduced in recent Terraform Cloud updates) override workspace-specific variables and non-priority variable sets. Use them sparingly for organization-wide overrides.

### 8c.4: Run Triggers

Run triggers automatically queue a run in a "downstream" workspace when a "source" workspace successfully applies. This enables infrastructure pipelines.

```hcl
# Workspace A (network) creates VPC and subnets.
# Workspace B (application) needs the VPC ID from A.

# In Workspace B, you configure a run trigger that targets Workspace A.
# Whenever Workspace A applies successfully, a new run is queued in Workspace B.
```

Run triggers are configured in the HCP Terraform UI or via API, not in Terraform configuration files.

### 8c.5: Health Assessments

Health assessments include:

- **Drift detection:** compares actual infrastructure to the state file.
- **Continuous validation:** re-evaluates `check` blocks and conditions.

They run automatically on a schedule (every 24 hours by default) and can be triggered on-demand.

---

### 8c.6: Hands-On Lab — Organizing with Projects and Workspaces (Conceptual)

Since this requires HCP Terraform admin permissions, we'll walk through the logical steps:

1. **Create projects:** In the UI, go to Projects → New Project. Create `platform` and `application` projects.
2. **Create workspaces:** Under the `platform` project, create a workspace `networking-prod`. Under `application`, create `app-prod`. Connect them to their respective VCS repositories.
3. **Configure variable sets:** Create a variable set `AWS Credentials` scoped to the organization (global). Create another `App Config` scoped to the `application` project with variables like `instance_type = "t3.medium"`.
4. **Set up run trigger:** In `app-prod`, add a run trigger from `networking-prod`. Now, when networking changes apply, `app-prod` automatically queues a run.
5. **Observe health assessments:** Enable assessments on `app-prod`. Trigger an on-demand assessment.

### 8c.7: Mini-Quiz for 8c

1. True/False: HCP Terraform workspaces function identically to CLI workspaces created with `terraform workspace new`.
2. Multiple Choice: What is the maximum scope of a variable set?
   A. A single workspace.
   B. A single project.
   C. The entire organization (global).
   D. A single Terraform run.
3. Multiple Answer: Which of the following features does HCP Terraform provide to organize workspaces? (Select TWO)
   A. Projects
   B. Modules
   C. Variable sets
   D. Data sources

---

<details>

<summary>Show Answers</summary>
1. False. HCP Terraform workspaces are fully independent and can have different configurations, variables, and execution modes. CLI workspaces are alternate state files within the same configuration directory.
2. C. A variable set can be scoped to a single workspace, a project, or the entire organization (global). There is no variable set scope limited to a single Terraform run.
3. A, C. HCP Terraform provides Projects to group workspaces and Variable Sets to manage variables across multiple workspaces. Modules and data sources are Terraform configuration concepts, not organizational features in HCP Terraform.

</details>

## Objective 8d: Configure and use HCP Terraform integration

*Source: `Connect to HCP Terraform`, `Migrate state to HCP Terraform`, `terraform login command`, `Dynamic provider credentials`*

### 8d.1: The CLI Integration (`cloud` block and `terraform login`)

We covered the `cloud` block in 8a. Let's reinforce the migration path.

**Scenario: Migrating from a local state to HCP Terraform.**

1. You have an existing Terraform project with a local state file.
2. Add the `cloud` block (as shown in 8a) with your organization and workspace name.
3. Run `terraform init`. Terraform detects the state migration and asks: "Do you want to copy existing state to the new cloud backend?" Type `yes`.
4. Your local state is uploaded to HCP Terraform. The local `terraform.tfstate` file is no longer used.
5. Run `terraform plan` and `terraform apply` — they now execute remotely.

### 8d.2: The `cloud` Block in Detail

```hcl
terraform {
  cloud {
    # 'hostname' is optional. Defaults to 'app.terraform.io'.
    # Set this to a custom value if using Terraform Enterprise.
    hostname = "app.terraform.io"

    # 'organization' is required.
    organization = "my-org"

    # 'workspaces' is required.
    workspaces {
      # Only one of 'name' or 'tags' can be specified.
      name = "my-workspace"
    }
  }
}
```

**Explanation:**

- `hostname = "app.terraform.io"` — The address of your HCP Terraform instance. For SaaS, this is always `app.terraform.io`. For Terraform Enterprise (self-hosted), you'd provide your company's custom hostname.
- `organization = "my-org"` — The unique name of your HCP Terraform organization.
- `workspaces { name = "my-workspace" }` — Specifies a single workspace. Terraform will create it during `init` if it doesn't exist.

**Environment variable overrides:**

Instead of hardcoding values, you can use environment variables. This allows the same configuration to be used across different organizations or Terraform Enterprise instances without changing the code.

| Environment Variable | Overrides |
|----------------------|-----------|
| `TF_CLOUD_ORGANIZATION` | `organization` |
| `TF_CLOUD_HOSTNAME` | `hostname` |
| `TF_WORKSPACE` | Workspace name when using tags |

**Example of environment-driven configuration:**
```hcl
# No hardcoded organization or workspace—these come from the environment.
terraform {
  cloud {
    # 'organization' is omitted—must be set via TF_CLOUD_ORGANIZATION env var.
    # 'workspaces' is omitted—individual workspace selected via TF_WORKSPACE.
    # An empty cloud block with only these overrides provides maximum flexibility.
  }
}
```

### 8d.3: Dynamic Provider Credentials

A major security feature. Instead of storing long-lived cloud credentials as workspace variables, you can configure a trust relationship between HCP Terraform and your cloud provider (AWS, Azure, GCP, Vault).

**How it works:**

1. You set up an OIDC trust relationship in your cloud provider.
2. HCP Terraform generates a workload identity token for each run.
3. The cloud provider verifies the token and issues short-lived credentials.
4. Terraform uses these credentials for the run.
5. The credentials expire after the run; no long-lived secrets are stored.

**Configuration example (workspace variables):**
```hcl
# No credentials in configuration! Set these as workspace environment variables:
TFC_AWS_PROVIDER_AUTH = 1
TFC_AWS_RUN_ROLE_ARN   = "arn:aws:iam::123456789:role/tfc-role"
```

**Explanation:**

- `TFC_AWS_PROVIDER_AUTH = 1` — Tells HCP Terraform to use dynamic credentials for the AWS provider.
- `TFC_AWS_RUN_ROLE_ARN` — The ARN of the IAM role that HCP Terraform should assume.

**Important note:** Dynamic credentials eliminate the need to store AWS access keys as workspace variables. They provide per-run, short-lived credentials.

### 8d.4: Excluding Files from Upload

When using remote execution, HCP Terraform uploads your configuration directory. You can create a `.terraformignore` file to exclude files (like `.git/`, `.terraform/`, local state files, secrets).

```txt
# .terraformignore
.git/
*.tfstate
*.tfvars
```

### 8d.5: Hands-On Lab — Configuring Dynamic Credentials (Conceptual)

We'll walk through the AWS setup conceptually.

1. **Create an OIDC provider in AWS** that trusts HCP Terraform's OIDC endpoint (`app.terraform.io`).
2. **Create an IAM role** with a trust policy that allows `app.terraform.io` to assume it, scoped to your organization and workspace.
3. **Attach an IAM policy** to the role granting permissions (e.g., EC2 full access).
4. **In HCP Terraform workspace variables**, set:
   - `TFC_AWS_PROVIDER_AUTH` = `1`
   - `TFC_AWS_RUN_ROLE_ARN` = the ARN of the IAM role.
5. **No AWS credentials** are stored as workspace variables. Terraform obtains temporary credentials for each run.

### 8d.6: API example — queue a run (curl)

You can queue runs programmatically via the HCP Terraform Runs API. Example (replace placeholders):

```bash
curl --header "Authorization: Bearer ${TFE_TOKEN}" \
  --header "Content-Type: application/vnd.api+json" \
  --data '{"data":{"attributes":{"is-destroy":false,"message":"API triggered run"},"type":"runs","relationships":{"workspace":{"data":{"type":"workspaces","id":"<WORKSPACE_ID>"}}}}}' \
  https://app.terraform.io/api/v2/runs
```

The API returns a run object; follow the `apply` status or use the Runs endpoint to poll for progress.

Reference: HCP Terraform API docs for runs.

### 8d.7: Migration commands & best practices

When migrating state to HCP Terraform or changing backends, prefer these safe steps:

1. Backup your local state: `cp terraform.tfstate terraform.tfstate.backup` (or download from remote backend).
2. Add the `cloud` block to your configuration.
3. Run `terraform init`: Terraform will prompt to copy state to the cloud backend.
4. Alternatively, use `terraform init -reconfigure` and `terraform init -migrate-state` to explicitly migrate state when changing backend configuration.
5. After migration, run `terraform plan` to validate.

Notes:
- Test migration on a copy of the workspace first.
- Ensure CI tokens and ACLs are correctly set for workspace access before migrating.

### 8d.8: Minimal policy example (OPA / Rego)

Here's a tiny Rego example that denies creation of an S3 bucket with `acl = "public-read"`:

```rego
package hcp.policies

deny[msg] {
  input.planned_changes[_].type == "aws_s3_bucket"
  some i
  change := input.planned_changes[_]
  change.change.after.acl == "public-read"
  msg = sprintf("S3 bucket %v would be public", [change.address])
}
```

For Sentinel, the equivalent policy would inspect the run's planned values and return `false` when a public ACL is detected. See HashiCorp docs for full examples.


---

### 8d.9: Mini-Quiz for 8d

1. True/False: The `cloud` block can coexist with a `backend` block in the same `terraform {}` configuration.
2. Multiple Choice: Which environment variable overrides the `organization` value in a `cloud` block?
   A. `TF_CLOUD_ORGANIZATION`
   B. `TF_ORGANIZATION`
   C. `TF_WORKSPACE`
   D. `TF_CLOUD_HOSTNAME`
3. Multiple Answer: Which of the following are required to use dynamic AWS credentials in HCP Terraform? (Select TWO)
   A. An IAM role with a trust relationship to HCP Terraform's OIDC endpoint.
   B. A `provider "aws"` block with hardcoded credentials.
   C. The workspace variable `TFC_AWS_PROVIDER_AUTH` set to `1`.
   D. An S3 backend configured for state storage.

---
<details>

<summary>Show Answers</summary>

1. False. The `cloud` block is mutually exclusive with a `backend` block. You cannot have both in the same configuration.
2. A. The `TF_CLOUD_ORGANIZATION` environment variable overrides the `organization` value in the `cloud` block. The other options are not correct for this purpose.
3. A, C. To use dynamic AWS credentials, you need an IAM role with a trust relationship to HCP Terraform's OIDC endpoint and the workspace variable `TFC_AWS_PROVIDER_AUTH` set to `1`. You do not need hardcoded credentials or an S3 backend for this feature.

</details>


## Objective 8: Quiz

**Question 1 (True/False):**
When you run `terraform apply` with a `cloud` block configured, the Terraform CLI executes the plan locally and only uploads the state to HCP Terraform.

**Question 2 (Multiple Choice):**

Which HCP Terraform feature allows you to define reusable collections of variables and apply them across multiple workspaces?
```
⬜ Variable sets
⬜ Projects
⬜ Policy sets
⬜ Run triggers
```
**Question 3 (Multiple Answer):**

Which of the following are valid ways to initiate a run in an HCP Terraform workspace? (Select **THREE**)
```
⬜ Merge a pull request to the main branch in a VCS-linked workspace.
⬜ Click "Start new run" in the workspace UI.
⬜ Run `terraform apply` on your local machine with a `cloud` block configured.
⬜ Use the HCP Terraform Runs API.
⬜ Run `terraform plan` on your local machine.
```
**Question 4 (True/False):**

HCP Terraform workspaces are identical to Terraform CLI workspaces and serve the same purpose of managing multiple state files for a single configuration.

**Question 5 (Multiple Choice):**

You want a policy that blocks any Terraform run from creating an S3 bucket with public read access. The policy should prevent the apply unless a security admin explicitly overrides it. What policy enforcement level should you use?
```
⬜ `advisory`
⬜ `hard-mandatory`
⬜ `soft-mandatory`
⬜ `warning`
```
**Question 6 (Multiple Choice):**

Which block is used to exclude certain files from being uploaded to HCP Terraform during remote execution?
```
⬜ `.gitignore`
⬜ `.terraformignore`
⬜ `terraform.ignore`
⬜ `exclude` block in `terraform {}`
```
**Question 7 (True/False):**

A project in HCP Terraform can contain multiple workspaces, and a user's permissions on a project automatically apply to all workspaces within it.

**Question 8 (Multiple Answer):**

Which of the following information does HCP Terraform's Explorer provide? (Select **TWO**)
```
⬜ A list of workspaces that have configuration drift.
⬜ The current market cost of all cloud resources.
⬜ The most commonly used provider versions across workspaces.
⬜ The exact source code of every module.
```
**Question 9 (Multiple Choice):**

What is the primary advantage of using dynamic provider credentials over static credentials stored as workspace variables?
```
⬜ They are easier to manually rotate.
⬜ They eliminate the need to store long-lived secrets in HCP Terraform.
⬜ They work with all Terraform providers without additional configuration.
⬜ They are automatically encrypted in state files.
```
**Question 10 (True/False):**

When migrating an existing local state file to HCP Terraform, you must first manually upload the state file via the HCP Terraform UI before running `terraform init`.

**Question 11 (Multiple Choice):**

A workspace `app-prod` has a run trigger configured to trigger from `networking-prod`. What causes a run to be queued in `app-prod`?
```
⬜ A successful `terraform plan` in `networking-prod`.
⬜ A successful `terraform apply` in `networking-prod`.
⬜ Any run in `networking-prod`, regardless of outcome.
⬜ A manual trigger from the `networking-prod` workspace UI.
```
**Question 12 (Multiple Answer):**

Which of the following are features of HCP Terraform's health assessments? (Select **TWO**)
```
⬜ Drift detection
⬜ Automatic resource rollback
⬜ Continuous validation
⬜ Cost estimation
```
**Question 13 (True/False):**

The `cloud` block supports using `tags` to associate a configuration with multiple workspaces instead of a single workspace name.

**Question 14 (Multiple Choice):**

Which environment variable must be set for HCP Terraform to automatically authenticate the AWS provider using dynamic credentials?
```
⬜ `AWS_ACCESS_KEY_ID`
⬜ `TFC_AWS_PROVIDER_AUTH`
⬜ `TFC_AWS_RUN_ROLE_ARN`
⬜ `AWS_DYNAMIC_CREDENTIALS`
```
**Question 15 (True/False):**

A workspace configured for "Local" execution mode still uses HCP Terraform for remote state storage and locking.

**Question 16 (Multiple Choice):**

You have a team of 5 developers who should be able to view workspace data but not modify it. You also have a team of 2 admins who need full control. How should you configure permissions?
```
⬜ Grant both teams "Admin" on each workspace.
⬜ Grant the developer team "Read" and the admin team "Admin" on each workspace.
⬜ Grant both teams "Write" on each workspace.
⬜ Only grant permissions at the organization level; workspace-level permissions are not supported.
```
**Question 17 (Multiple Answer):**

Which of the following can be configured in a `.terraformignore` file? (Select **TWO**)
```
⬜ Patterns for files to exclude from upload.
⬜ Backend configuration.
⬜ Rules based on `.gitignore` syntax.
⬜ Variable definitions.
```
**Question 18 (True/False):**

When using HCP Terraform's CLI-driven workflow, `terraform plan` creates a speculative plan that does not lock the workspace or queue a run.

**Question 19 (Multiple Choice):**

What is the function of the `TF_WORKSPACE` environment variable when using the `cloud` block with tags?
```
⬜ It overrides the `hostname` setting.
⬜ It selects which tagged workspace to use for the current operation.
⬜ It defines a new tag to apply to the workspace.
⬜ It sets the execution mode of the workspace.
```
**Question 20 (True/False):**

A priority variable set in HCP Terraform takes precedence over a workspace-specific variable of the same name.

---

<details>

<summary>Show Answers</summary>

**Question 1:** True - With the `cloud` block configured, `terraform apply` queues a run that executes remotely on HCP Terraform's infrastructure. The plan is generated and applied on HCP Terraform's servers, not locally.
**Question 2:** False - The correct answer is "Variable sets". Projects are for grouping workspaces, policy sets are for grouping policies, and run triggers are for automating runs between workspaces.
**Question 3:** True - All of these can initiate a run. Pushing code to a linked VCS repository, clicking "Start new run" in the UI, running `terraform apply` locally with a `cloud` block configured, and using the HCP Terraform API can all trigger runs. Running `terraform plan` does not trigger a run; it generates a speculative plan locally.
**Question 4:** False - HCP Terraform workspaces are fully independent and can have different configurations, variables, and execution modes. CLI workspaces are alternate state files within the same configuration directory.
**Question 5:** True - The `soft-mandatory` enforcement level prevents a run from being applied unless an authorized user overrides the failure. `hard-mandatory` would block the run entirely without an override option, and `advisory` would only show a warning.
**Question 6:** False - The correct answer is `.terraformignore`. This file is used to specify patterns for files that should be excluded from being uploaded to HCP Terraform during remote execution. `.gitignore` is for Git, and the other options are not valid for this purpose.
**Question 7:** True - A project in HCP Terraform can contain multiple workspaces, and a user's permissions on a project automatically apply to all workspaces within it. This allows for easier management of permissions across related workspaces.
**Question 8:** A, C - The Explorer provides information such as a list of workspaces that have configuration drift and the most commonly used provider versions across workspaces. It does not provide market cost information or the exact source code of every module.
**Question 9:** B - The primary advantage of using dynamic provider credentials is that they eliminate the need to store long-lived secrets in HCP Terraform. Dynamic credentials are short-lived and automatically rotated, enhancing security. They do require additional configuration and are not automatically compatible with all providers. They are also encrypted in state files, but the key advantage is the elimination of long-lived secrets.
**Question 10:** False - When migrating an existing local state file to HCP Terraform, you do not need to manually upload the state file. During `terraform init`, Terraform detects the state migration and prompts you to copy the existing state to the new cloud backend. You can simply confirm this action, and Terraform will handle the upload for you.
**Question 11:** B - A run is queued in `app-prod` when there is a successful `terraform apply` in `networking-prod`. A successful `terraform plan` does not trigger a run, and runs are not triggered by any run in `networking-prod` regardless of outcome or by manual triggers from the UI.
**Question 12:** A, C - HCP Terraform's health assessments include drift detection and continuous validation. They do not include automatic resource rollback or cost estimation as part of the health assessment features.
**Question 13:** True - The `cloud` block supports using `tags` to associate a configuration with multiple workspaces. This allows the same configuration to apply to any workspace that has the specified tags, providing flexibility in how you organize and manage your workspaces.
**Question 14:** C - The environment variable `TFC_AWS_RUN_ROLE_ARN` must be set to specify the ARN of the IAM role that HCP Terraform should assume for dynamic AWS credentials. `TFC_AWS_PROVIDER_AUTH` is also required to enable dynamic credentials, but it does not specify the role ARN. The other options are not correct for this purpose.
**Question 15:** True - A workspace configured for "Local" execution mode still uses HCP Terraform for remote state storage and locking. The difference is that the plan and apply operations execute locally on the user's machine instead of remotely on HCP Terraform's infrastructure. However, state management and locking still occur through HCP Terraform to ensure consistency and prevent conflicts.
**Question 16:** B - The correct configuration is to grant the developer team "Read" and the admin team "Admin" on each workspace. This allows developers to view workspace data without modifying it, while admins have full control. Granting both teams "Admin" would give developers more permissions than necessary, and granting both teams "Write" would allow developers to modify workspace data, which is not desired. Permissions can be granted at both the organization and workspace levels, but workspace-level permissions are necessary for this scenario.
**Question 17:** A, C - A `.terraformignore` file can be configured with patterns for files to exclude from upload and can use rules based on `.gitignore` syntax. It does not contain backend configuration or variable definitions, which are specified in Terraform configuration files.
**Question 18:** False - When using HCP Terraform's CLI-driven workflow, `terraform plan` creates a speculative plan that does not lock the workspace or queue a run. This allows you to see what would happen without affecting the state or triggering any actions in HCP Terraform. Only `terraform apply` queues a run and locks the workspace for execution.
**Question 19:** B - The `TF_WORKSPACE` environment variable is used to select which tagged workspace to use for the current operation when using the `cloud` block with tags. It allows you to specify the workspace context without hardcoding it in the configuration, providing flexibility in multi-workspace setups. It does not override the `hostname`, define new tags, or set the execution mode.
**Question 20:** True - A priority variable set in HCP Terraform takes precedence over a workspace-specific variable of the same name. This means that if there is a conflict between a priority variable set and a workspace-specific variable, the value from the priority variable set will be used in the Terraform run. This allows for organization-wide overrides of variables when necessary.

</details>

---

You did it! Take this moment to really celebrate. You have completed Objective 8, which means you have a solid understanding of how to use HCP Terraform for collaboration, governance, and remote execution. This is a major milestone in your Terraform journey.

Thank you for letting me be your study guide throughout this entire course. I hope you found it helpful, engaging, and maybe even a little fun. Remember, the best way to learn is by doing, so keep practicing with HCP Terraform and applying what you've learned in real-world scenarios.

If you are still unclear on any objective, feel free to go back and review the sections, watch the videos, and try the hands-on labs again. The more you interact with the material, the more it will stick. And when you feel ready, you can take the official HashiCorp Terraform Certification exam to validate your knowledge and skills.

> Congratulations again on completing Objective 8, and best of luck on your [certification journey!](./End.md)