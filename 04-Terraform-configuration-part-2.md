# Objective 4 (Part-2): Write and understand Terraform configuration


## Part 2: this is where the magic happens

This is the heart of Terraform configuration. The first half of Objective 4 covers the basics of resources, data sources, variables, outputs, and complex types. This second half dives into the more advanced features that make Terraform truly powerful and flexible.

- [Objective 4e: Write dynamic configuration using expressions and functions](#objective-4e-write-dynamic-configuration-using-expressions-and-functions)

- [Objective 4f: Define resource dependencies in configuration](#objective-4f-define-resource-dependencies-in-configuration)

- [Objective 4g: Validate configuration using custom conditions](#objective-4g-validate-configuration-using-custom-conditions)

- [Objective 4h: Understand best practices for managing sensitive data](#objective-4h-understand-best-practices-for-managing-sensitive-data)

- [Objective 4: Quiz](#objective-4-part-2-quiz-functions-expressions-and-validation)

> These are the advanced configuration language features. They are not required for simple use cases, but they are essential for building robust, maintainable, and secure infrastructure as code. Mastering these will set you apart as a Terraform practitioner and prepare you for the more complex scenarios covered in later objectives.

## Objective 4e: Write dynamic configuration using expressions and functions

*Source: [`Perform dynamic operations with functions`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/functions), [`Built-in Functions`](https://developer.hashicorp.com/terraform/language/v1.12.x/functions), [`Create dynamic expressions`](https://developer.hashicorp.com/terraform/language/v1.12.x/expressions)*

### 4e.1: Expressions Overview

Terraform expressions allow you to compute values dynamically rather than hardcoding them. The guide distinguishes:

- **Simple values:** `"hello"`, `42`, `true`
- **References:** `var.name`, `aws_instance.web.id`
- **Operators:** `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `>`, `<`, `&&`, `||`, `!`
- **Conditionals:** `condition ? true_val : false_val`
- **For expressions:** `[for s in var.list : upper(s)]`
- **Splat expressions:** `aws_instance.web[*].id`
- **String templates:** `"Hello, ${var.name}!"`
- **Function calls:** `upper("hello")`

Note on types and errors:

- Terraform attempts safe type coercion where sensible (for example, converting numeric strings to numbers in some functions), but you should not rely on implicit conversions in complex expressions. Prefer explicit conversion functions like `tostring()`, `tonumber()`, and `tolist()` when the type matters.
- Use `can()` to test whether an expression will succeed without causing a plan-time error. Example: `can(regex("^v[0-9]+", var.version))`.
- Use `try(expr, fallback)` to attempt expressions that may fail and provide a fallback value. This helps avoid runtime errors in `for` expressions or optional attributes.

>**Best practice:** keep expressions readable and predictable. If a single expression becomes too complex, compute intermediate values with `locals {}` and give them descriptive names.

### 4e.2: Conditional Expressions

Syntax: `condition ? value_if_true : value_if_false`

```hcl
resource "aws_instance" "web" {
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
}
```

**Key Points:**
- Both result values must be of the **same type** (or convertible).
- The condition must evaluate to a boolean.
- Conditional expressions can be nested, but readability suffers.

### 4e.3: For Expressions

For expressions transform lists, sets, tuples, maps, or objects into new collections.

**List transformation:**
```hcl
[for s in var.names : upper(s)]
# Input: ["alice", "bob"] -> Output: ["ALICE", "BOB"]
```

**Map transformation:**
```hcl
{for k, v in var.tags : k => upper(v)}
# Input: {env = "dev"} -> Output: {env = "DEV"}
```

**Filtering with `if`:**
```hcl
[for s in var.names : upper(s) if length(s) > 3]
```

**Grouping with `...` (ellipsis):**
```hcl
{for s in var.servers : s.name => s...}
# Creates a map keyed by name with the full object as value
```

### 4e.4: Splat Expressions

The splat operator `[*]` extracts a single attribute from all elements of a list.

```hcl
resource "aws_instance" "web" {
  count = 3
  # ...
}

output "instance_ids" {
  value = aws_instance.web[*].id
}
```

For maps (from `for_each`), use `values()` first:
```hcl
values(aws_instance.web)[*].id
```

**Legacy splat:** `aws_instance.web.*.id` — still works but `[*]` is preferred.

### 4e.5: String Templates and Heredocs

**Interpolation:**
```hcl
"Hello, ${var.name}!"
```

**Directives:** (Terraform 1.6+)
```hcl
"%{if condition}text%{else}other%{endif}"
```

**Heredoc syntax:**
```hcl
<<-EOT
  This is a multi-line string.
  Variables like ${var.name} are interpolated.
  EOT
```

### 4e.6: Built-in Functions

The offical study guide lists **dozens** of functions. The exam focuses on common ones and their use cases.

**Numeric:**
- `max(1, 5, 3)` → `5`
- `min(...)`
- `ceil(3.14)` → `4`
- `floor(3.14)` → `3`

**String:**
- `upper("hello")` → `"HELLO"`
- `lower("HELLO")` → `"hello"`
- `join(", ", ["a", "b"])` → `"a, b"`
- `split(",", "a,b,c")` → `["a", "b", "c"]`
- `replace("hello world", "world", "Terraform")` → `"hello Terraform"`
- `substr("hello", 1, 3)` → `"ell"`
- `trimspace("  hi  ")` → `"hi"`
- `regex("^[a-z]+", "abc123")` → `"abc"`

**Collection:**
- `length([1,2,3])` → `3`
- `lookup(map, key, default)`
- `merge(map1, map2)`
- `concat(list1, list2)`
- `element(list, index)`
- `slice(list, start, end)`
- `keys(map)` → list of keys
- `values(map)` → list of values
- `contains(list, "value")` → boolean
- `flatten(list_of_lists)`
- `setintersection(set1, set2)`
- `setunion(set1, set2)`
- `toset(list)` → removes duplicates

**Filesystem:**
- `file("path")` → returns file contents as string
- `templatefile("template.tpl", { var = value })` → renders template
- `filebase64("path")` → base64 encoded content

**Encoding:**
- `jsonencode({ key = "value" })` → `{"key":"value"}`
- `jsondecode("...")`
- `yamlencode(...)`
- `yamldecode(...)`
- `base64encode(...)`
- `base64decode(...)`

**Date and Time:**
- `timestamp()` → current UTC timestamp
- `formatdate("YYYY-MM-DD", timestamp())`
- `timeadd(timestamp(), "24h")`

**IP and CIDR:**
- `cidrsubnet("10.0.0.0/16", 8, 1)` → `"10.0.1.0/24"`
- `cidrhost("10.0.0.0/24", 5)` → `"10.0.0.5"`
- `cidrnetmask("10.0.0.0/24")` → `"255.255.255.0"`

**Type Conversion:**
- `tostring(value)`
- `tonumber(value)`
- `tobool(value)`
- `tolist(set)`
- `tomap(object)`

#### Common pitfalls and best practices:

- Beware of functions that return different types for different inputs (e.g., `jsondecode()` may produce maps, lists, or scalars). Explicitly validate or guard with `can()` when needed.
- Avoid heavy use of nested function calls in resource arguments; compute intermediate values in `locals {}` for clarity and easier debugging.
- Prefer `lookup(map, key, default)` over direct indexing when keys may be absent to avoid plan-time errors.
- Use `try()` to provide fallbacks for optional or provider-specific attributes that may not exist in all contexts.

### 4e.7: The `templatefile` Function

This is a **critical exam topic**. It reads a template file and interpolates variables into it.

**Template file (`user_data.tpl`):**
```bash
#!/bin/bash
echo "Server name: ${server_name}" >> /var/log/startup.log
echo "Environment: ${environment}" >> /var/log/startup.log
```

**Usage in Terraform:**
```hcl
resource "aws_instance" "web" {
  user_data = templatefile("${path.module}/user_data.tpl", {
    server_name = var.name
    environment = var.env
  })
}
```

**Key Points:**
- Template variables use `${...}` syntax (like Terraform, but separate context).
- The second argument is a map of variable values.
- The result is a string that can be passed to resource arguments.

### 4e.8: Provider-Defined Functions

Some providers expose custom functions (Terraform 1.8+). Syntax:

```hcl
provider::aws::arn_parse("arn:aws:s3:::mybucket")
```

The exam may test recognition that functions can come from providers, not just built-in.

### 4e.9: Hands-On Lab — Functions and Expressions

**Directory:** `obj4e-lab`

**File: `variables.tf`**

```hcl
variable "servers" {
  type = list(object({
    name = string
    size = string
  }))
  default = [
    { name = "web", size = "small" },
    { name = "db", size = "large" },
    { name = "cache", size = "medium" }
  ]
}
```

**File: `template.tpl`**
```
Servers:
%{ for s in servers ~}
- ${s.name} (${s.size})
%{ endfor ~}
Generated at: ${timestamp}
```

**File: `main.tf`**

```hcl
terraform {
  required_providers {
    local = { source = "hashicorp/local", version = "~> 2.5" }
  }
}

locals {
  large_servers = [for s in var.servers : s.name if s.size == "large"]
  server_names  = join(", ", [for s in var.servers : upper(s.name)])
}

resource "local_file" "report" {
  filename = "${path.module}/report.txt"
  content  = templatefile("${path.module}/template.tpl", {
    servers   = var.servers
    timestamp = formatdate("YYYY-MM-DD HH:mm", timestamp())
  })
}

output "large_servers" {
  value = local.large_servers
}

output "all_server_names_uppercase" {
  value = local.server_names
}
```

**Steps:**
1. `terraform init`
2. `terraform plan` — Observe computed values.
3. `terraform apply -auto-approve`
4. Inspect `report.txt` and outputs.
5. Experiment with `terraform console` to test functions interactively:

   ```bash
   terraform console
   > upper("test")
   "TEST"
   > max(1,5,3)
   5
   > slice(["a","b","c","d"], 1, 3)
   ["b", "c"]
   > cidrsubnet("10.0.0.0/16", 8, 2)
   "10.0.2.0/24"
   ```

### 4e.10: Mini-Quiz - Functions, Expressions, and Validation

1. True/False: The `templatefile` function can only be used with files ending in `.tpl`.

2. Multiple Choice: Which function would you use to combine two maps into one?

```
   A. `concat`
   B. `join`
   C. `merge`
   D. `union`
   ```

3. Multiple Answer: Which of the following are valid Terraform expressions? (Select TWO)

```
   A. `"Hello, ${var.name}"`
   B. `[for x in var.list : x if x > 0]`
   C. `if var.env == "prod" then "large" else "small"`
   D. `var.tags["Name"]`
```

<details>

<summary>Answers</summary>

**Answers & brief explanations**

1. False — `templatefile` can read any file, the extension does not matter. The `.tpl` convention is just for clarity.
2. C — `merge` combines maps. `concat` is for lists, `join` is for strings, and `union` is for sets.
3. A, B — A is a valid string interpolation, and B is a valid for expression with filtering. C is invalid syntax (should use `? :`), and D is valid only if `var.tags` is a map, but the question asks for expressions in general, so we cannot assume that.

</details>

## Objective 4f: Define resource dependencies in configuration

*Source: [`Create resource dependencies`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/dependencies), [`Resource graph`](https://developer.hashicorp.com/terraform/internals/v1.12.x/graph)*

### 4f.1: Dependency Graph Fundamentals

Terraform automatically builds a **dependency graph** by analyzing resource references. The guide's `Dependency Graph` section details:

- **Nodes:** Resources, data sources, providers.
- **Edges:** Dependencies between nodes.
- **Walk:** Terraform traverses the graph in parallel, respecting dependencies.

**Implicit Dependencies:** Created when a resource references another resource's attribute.
```hcl
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "main" {
  vpc_id = aws_vpc.main.id  # Implicit dependency
}
```

**Explicit Dependencies:** Use `depends_on` when Terraform cannot infer the relationship.
```hcl
resource "aws_s3_bucket" "data" { ... }
resource "aws_instance" "app" {
  depends_on = [aws_s3_bucket.data]  # Explicit
}
```

### 4f.2: When to Use `depends_on`

The guide lists specific scenarios:
- When a resource needs to wait for a side effect that isn't captured in attributes (e.g., an IAM role propagation delay).
- When a resource depends on an **entire module** rather than a specific output.
- When you want to force ordering for application-level reasons.

#### Nuances for modules and cross-module references:

- Modules do not implicitly share internal resources. If Module A creates something Module B needs, the common pattern is Module A exposes an output and the root (or caller) passes that output into Module B. Alternatively, you can set `depends_on = [module.module_a]` in Module B's module block when the ordering must be enforced for the whole module.
- `depends_on` in a module block applies at the module-call site (root module) and ensures the entire child module waits for the referenced modules/resources.

Performance and maintenance caution:

- Each `depends_on` edge can reduce concurrency; overusing it serializes applies and slows teams down. Use explicit `depends_on` only when the dependency is not otherwise represented by references or outputs.

**Module-Level `depends_on`:**
```hcl
module "vpc" { ... }
module "app" {
  depends_on = [module.vpc]
}
```

**Important Note:** Overusing `depends_on` reduces parallelism and slows down applies. Only use when necessary.

### 4f.3: Lifecycle Meta-Argument

The `lifecycle` block inside a `resource` controls how Terraform manages that resource's lifecycle.

```hcl
resource "aws_instance" "web" {
  # ... arguments ...

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes        = [tags, user_data]
    replace_triggered_by  = [aws_security_group.allow_ssh.id]
  }
}
```

**Lifecycle Rules:**

- **`create_before_destroy` (bool):** When a resource must be replaced, create the new one **before** destroying the old one. Reduces downtime.

Example (avoid downtime for a database failover):

```hcl
resource "aws_db_instance" "master" {
  # ... args ...

  lifecycle {
    create_before_destroy = true
  }
}
```

Trade-offs: `create_before_destroy` may temporarily increase costs (two instances) and may not be possible when the provider or resource enforces unique names. In those cases, consider designing for immutable resources with blue/green patterns at the application layer.

- **`prevent_destroy` (bool):** If `true`, Terraform will error if a plan would destroy this resource. Protects critical resources.
- **`ignore_changes` (list):** Terraform will ignore differences in the listed attributes when planning updates.
- **`replace_triggered_by` (list):** Forces replacement of this resource if any of the referenced resources or attributes change.

`replace_triggered_by` is useful when a change in a dependent resource should cause a replacement of the consuming resource even if Terraform's default diff would not. Use it sparingly to keep replacements intentional.

### 4f.4: Hands-On Lab — Dependencies and Lifecycle

**Directory:** `obj4f-lab`

**File: `main.tf`**
```hcl
resource "random_pet" "first" {
  length = 1
}

resource "random_pet" "second" {
  length = 1
  depends_on = [random_pet.first]  # Explicit dependency
}

resource "local_file" "config" {
  filename = "${path.module}/config.txt"
  content  = "first: ${random_pet.first.id}, second: ${random_pet.second.id}"

  lifecycle {
    ignore_changes = [content]  # Changes to content won't trigger update
  }
}

output "first_pet" {
  value = random_pet.first.id
}

output "second_pet" {
  value = random_pet.second.id
}
```

**Steps:**

1. `terraform init && terraform apply -auto-approve`
2. Manually edit `config.txt` content. Run `terraform plan` — no changes detected due to `ignore_changes`.
3. Change the length of `random_pet.first` to `2`. Run `terraform plan`. Terraform will destroy and recreate `first`, then `second` (due to `depends_on`), but `local_file.config` will **not** update because of `ignore_changes`.
4. Add `prevent_destroy = true` to `local_file.config`. Try to `terraform destroy` — it will error.

### 4f.5: Mini-Quiz - Resource Dependencies and Lifecycle

1. True/False: `depends_on` can be used inside a `data` block.
2. Multiple Choice: Which lifecycle rule prevents accidental deletion of a resource?
   A. `create_before_destroy`
   B. `prevent_destroy`
   C. `ignore_changes`
   D. `replace_triggered_by`
3. Multiple Answer: When does Terraform automatically create an implicit dependency? (Select TWO)
   A. When a resource references another resource's attribute.
   B. When two resources share the same `provider` block.
   C. When a resource uses a data source that references a managed resource.
   D. When resources are defined in the same `.tf` file.

<details>

<summary>Answers</summary>

**Answers & brief explanations**

1. True — `depends_on` can be used in `data` blocks to specify dependencies on resources or other data sources.
2. B — `prevent_destroy` will cause Terraform to error if an operation would destroy the resource, providing a safeguard against accidental deletion.
3. A, C — An implicit dependency is created when a resource references another resource's attribute (A) or when a data source references a managed resource (C). Sharing the same provider block or being in the same file does not create dependencies.

</details>


## Objective 4g: Validate configuration using custom conditions

*Source: [`Validate your configuration`](https://developer.hashicorp.com/terraform/language/v1.12.x/validate), [`Validate modules with custom conditions`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/custom-conditions), [`Use checks to validate infrastructure`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/checks)*

### 4g.1: Types of Validation

Terraform offers multiple layers of validation:

| Type | Scope | When Evaluated | Blocks Operation? |
|------|-------|----------------|-------------------|
| Variable Validation | Input variables | During plan | Yes |
| Preconditions | Resources, data sources, outputs | Before creating/reading | Yes |
| Postconditions | Resources, data sources | After creating/reading | Yes |
| Checks | Any assertions | End of plan/apply | No (warning only) |

### 4g.2: Variable Validation

```hcl
variable "instance_type" {
  type = string

  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Only t3 instance types are allowed."
  }
}
```

**Multiple validations per variable are allowed.**

### 4g.3: Preconditions and Postconditions

These are defined within a `lifecycle` block for resources and data sources, or directly in `output` blocks.

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  lifecycle {
    # Check BEFORE creating the instance
    precondition {
      condition     = data.aws_ami.ubuntu.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }

    # Check AFTER creating the instance
    postcondition {
      condition     = self.public_ip != null
      error_message = "Instance must have a public IP."
    }
  }
}
```

**Output Precondition:**
```hcl
output "bucket_name" {
  value = aws_s3_bucket.data.bucket

  precondition {
    condition     = length(var.bucket_name) <= 63
    error_message = "Bucket name must be 63 characters or less."
  }
}
```

### 4g.4: Check Blocks

Checks validate infrastructure without blocking operations. They run at the **end** of `plan` and `apply`.

```hcl
check "website_health" {
  data "http" "index" {
    url = "https://${aws_instance.web.public_dns}"
  }

  assert {
    condition     = data.http.index.status_code == 200
    error_message = "Website returned ${data.http.index.status_code}"
  }
}
```

**Key Points:**

- `check` blocks are **not** tied to a specific resource lifecycle.
- They can contain their own `data` sources, scoped only to the check.
- Failures produce **warnings**, not errors.

Execution timing and CI guidance:

- **Preconditions** are evaluated before creating or reading the resource and will block `apply` when they fail. **Postconditions** run after the resource is created/read and can also block on failure.
- **Check blocks** run at the end of `plan` and `apply` and produce warnings (unless your CI treats warnings as failures). Use checks for observability and non-blocking alerts; use pre/postconditions for hard safety gates.
- In CI pipelines, run `terraform plan` in a pre-merge job and treat failed preconditions as a gate. Run periodic `terraform apply` or the HCP Terraform checks runner for continuous validation against drift.

### 4g.5: Continuous Validation in HCP Terraform

When enabled, HCP Terraform periodically evaluates `check` blocks and conditions in a workspace, alerting you to configuration drift or condition failures **without** requiring a new `apply`. This is covered in Objective 8.

### 4g.6: Hands-On Lab — Custom Conditions

**Directory:** `obj4g-lab`

**File: `variables.tf`**
```hcl
variable "file_name" {
  type = string

  validation {
    condition     = can(regex("^[a-z]+\\.txt$", var.file_name))
    error_message = "File name must be lowercase letters only and end with .txt"
  }
}

variable "content_length" {
  type    = number
  default = 10
}
```

**File: `main.tf`**
```hcl
resource "local_file" "data" {
  filename = "${path.module}/${var.file_name}"
  content  = random_string.generator.result

  lifecycle {
    postcondition {
      condition     = length(self.content) == var.content_length
      error_message = "Content length must be ${var.content_length}, got ${length(self.content)}"
    }
  }
}

resource "random_string" "generator" {
  length = var.content_length
}

check "file_exists" {
  data "local_file" "check" {
    filename = local_file.data.filename
  }

  assert {
    condition     = fileexists(data.local_file.check.filename)
    error_message = "File ${data.local_file.check.filename} was not created."
  }
}
```

**Steps:**

1. `terraform init`
2. Try `terraform plan -var="file_name=BAD.txt"` — validation fails.
3. Use valid name: `terraform apply -var="file_name=good.txt" -auto-approve`
4. Observe postcondition passes.
5. Manually delete the file. Run `terraform plan` — the `check` block will still pass because it uses the data source defined within it (which reads the current state? Actually, the data source in the check will re-evaluate during plan and fail because file doesn't exist). This demonstrates checks catching drift.
5. Manually delete the file. Run `terraform plan` — the `check` block's `data` source is evaluated during planning for the check and will fail its assertion because the file no longer exists. This demonstrates checks catching drift at plan/apply time.

### 4g.7: Mini-Quiz - Custom Conditions

1. True/False: A failed precondition will stop the `terraform apply` from proceeding.

2. Multiple Choice: Which validation type produces a warning rather than an error when it fails?
```
   A. Variable validation
   B. Precondition
   C. Postcondition
   D. Check block assertion
   ```
3. Multiple Answer: Where can you define a precondition? (Select TWO)
```
   A. In a `variable` block
   B. In a `resource` lifecycle block
   C. In a `data` lifecycle block
   D. In a `provider` block
```
<details>

<summary>Answers</summary>

**Answers & brief explanations**

1. True — A failed precondition will cause Terraform to error and stop the apply process.
2. D — Check block assertions produce warnings when they fail, while variable validation, preconditions
and postconditions produce errors.
3. B, C — Preconditions can be defined in the lifecycle block of both `resource` and `data` blocks. They cannot be defined in `variable` or `provider` blocks.

</details>

## Objective 4h: Understand best practices for managing sensitive data

*Source: [`Protect sensitive input variables`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/sensitive-variables#sensitive-values-in-state), [`Manage sensitive data in your configuration`](https://developer.hashicorp.com/terraform/language/v1.12.x/manage-sensitive-data), [`Inject secrets into Terraform using the Vault provider`](https://developer.hashicorp.com/terraform/tutorials/secrets/secrets-vault)*

### 4h.1: The State of Sensitive Data in Terraform

Sensitive data (API keys, passwords, tokens) must be handled carefully. Terraform stores **all** resource attributes in the **state file in plaintext**. Even if you mark a variable `sensitive`, the value is **still in state**.

### 4h.2: Marking Variables and Outputs as Sensitive

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "db_password" {
  value     = aws_db_instance.master.password
  sensitive = true
}
```

**Effects:**

- Values are **redacted** in CLI output (`terraform plan`, `terraform apply`).
- They are **not** displayed in HCP Terraform UI.
- They **remain in plaintext in the state file**.

### 4h.3: Ephemeral Values (Terraform 1.10+)

The `ephemeral` argument prevents a value from being stored in state **at all**.

```hcl
variable "db_password" {
  type      = string
  ephemeral = true
}
```

**Restrictions:**

- Can only be used in specific contexts (provider configurations, provisioners, connection blocks).
- Cannot be referenced in normal resource arguments that persist to state.

### 4h.4: Write-Only Arguments (Terraform 1.11+)

Providers can define arguments with `_wo` suffix that are **write-only**. They are used to pass secrets without storing them.

```hcl
resource "aws_db_instance" "master" {
  password_wo = var.db_password
}
```

The value is used for the API call but **not stored in state**.

### 4h.5: Best Practices Summary

| Method | State Protection | CLI Redaction | Notes |
|--------|------------------|---------------|-------|
| `sensitive = true` | No | Yes | Still in state plaintext |
| `ephemeral = true` | Yes (excluded) | N/A | Limited usage contexts |
| Write-only args | Yes (excluded) | N/A | Provider-specific |
| Remote state encryption | Yes (encrypted) | N/A | S3 SSE, HCP Terraform Vault |
| Vault Provider | Yes (dynamic) | Yes | Generates short-lived credentials |

#### Additional recommendations:

- Store state in a remote backend that supports server-side encryption and access controls (e.g., `S3 with SSE-KMS` and `DynamoDB locking`, `Azure Storage` with encryption, or HCP Terraform). Use least-privilege IAM roles for the backend and enable versioning on buckets to recover from accidental changes.
- Never commit plaintext secret files or `.tfvars` containing secrets into source control. Use environment variables, CI secret stores, or SOPS/age-encrypted files that are decrypted during CI runs.
- In CI, avoid printing sensitive variables; use the CI provider's secret masking features and run `terraform plan` with sensitive values provided via secure variables or a remote state provider.
- When using Vault, prefer short-lived dynamic credentials via provider data sources rather than long-lived static tokens. Rotate secrets regularly and scope roles narrowly.
- Consider additional controls like encrypting state at rest, restricting who can run `terraform apply`, and using policy checks (Sentinel/Opa/Run Tasks) to prevent risky changes.

### 4h.6: The Vault Provider Pattern

The guide's `Inject secrets into Terraform using the Vault provider` tutorial describes a **dynamic credential** workflow.

```hcl
provider "vault" {
  address = "https://vault.example.com"
}

data "vault_aws_access_credentials" "creds" {
  backend = "aws"
  role    = "terraform-role"
}

provider "aws" {
  access_key = data.vault_aws_access_credentials.creds.access_key
  secret_key = data.vault_aws_access_credentials.creds.secret_key
}
```

**Benefits:**

- Credentials are **short-lived** (TTL).
- No long-lived keys stored in Terraform state.
- Vault manages the lifecycle of AWS IAM users dynamically.

### 4h.7: Hands-On Lab — Sensitive Data Handling (No Vault, Local Simulation)

**Directory:** `obj4h-lab`

**File: `variables.tf`**
```hcl
variable "secret_message" {
  type      = string
  sensitive = true
}
```

**File: `main.tf`**
```hcl
resource "local_file" "secret" {
  filename = "${path.module}/secret.txt"
  content  = var.secret_message
}

output "file_content" {
  value     = local_file.secret.content
  sensitive = true
}

output "file_path" {
  value = local_file.secret.filename
}
```

**Steps:**

1. `terraform init`
2. `terraform plan -var="secret_message=SuperSecret123"`
   - Observe `secret_message` is `(sensitive value)` in plan.
3. `terraform apply -var="secret_message=SuperSecret123" -auto-approve`
   - Output `file_content` is redacted.
4. Run `terraform output file_content` — value is displayed (because specific query bypasses redaction).
5. Inspect `terraform.tfstate` — search for `SuperSecret123`. It's there in plaintext.
6. Run `terraform output -json` — `file_content` is exposed in JSON output.

**Key Lesson:** Sensitive is UI redaction only. State file security is paramount.

--

Light module-nesting joke:

> Modules have feelings too — especially when nested. "Do you love me, parent module?" asks the child. The parent replies, "I reference you deeply, but please don't create a circular dependency!" 😄

### 4h.8: Mini-Quiz - Managing Sensitive Data

1. True/False: Marking a variable as `sensitive` prevents its value from being written to the state file.

2. Multiple Choice: Which Terraform feature ensures a value is never stored in state?

```
A. sensitive = true
B. ephemeral = true
C. sensitive on outputs
D. .gitignore
```

3. Multiple Answer: What are best practices for managing sensitive data in Terraform? **(Select TWO)**

```
A. Store state files in encrypted remote backends.
B. Commit .tfvars files to version control.
C. Use dynamic credentials from Vault instead of static keys.
D. Hardcode secrets in locals blocks.
```

<details>

<summary>Show Answers</summary>

**Answers & brief explanations**

1. **False** `sensitive` only redacts values in CLI output, but they are still stored in plaintext in the state file.
2. **(B)**`ephemeral = true` prevents the value from being stored in state at all. `sensitive` does not prevent state storage, and `.gitignore` is for ignoring files in version control, not state management.
3. **(A, C)** Storing state in encrypted remote backends and using dynamic credentials from Vault are best practices for managing sensitive data. Committing `.tfvars` files or hardcoding secrets in `locals` is not recommended.

</details>

## Objective 4 Part 2 Quiz: Functions, Expressions, and Validation

**Note down your answers, and check them against the provided solutions. (No peeping!)**


**Question 1 (True/False):**

The expression `[for s in ["a", "b"] : upper(s)]` returns `["A", "B"]`.

**Question 2 (Multiple Choice):**
Which function would you use to render a template file with variable substitution?
```
⬜ `file()`
⬜ `templatefile()`
⬜ `jsonencode()`
⬜ `format()`
```
**Question 3 (Multiple Answer):**

Which of the following are valid lifecycle rules? (Select THREE)
```
⬜ `create_before_destroy`
⬜ `prevent_apply`
⬜ `ignore_changes`
⬜ `replace_triggered_by`
⬜ `timeout`
```

**Question 4 (True/False):**

The `depends_on` meta-argument can be used inside a `module` block to ensure the module's resources are created after another module's resources.

**Question 5 (Multiple Choice):**

You have a map variable `var.tags = { env = "dev", owner = "team" }`. Which expression correctly retrieves the value of the `env` key?
```
⬜ `var.tags[0]`
⬜ `var.tags["env"]`
⬜ `lookup(var.tags, "env")`
⬜ Both B and C
```
**Question 6 (Multiple Choice):**

A precondition defined in a `resource` lifecycle block is evaluated:
```
⬜ After the resource is created.
⬜ Before the resource is created.
⬜ Only during `terraform validate`.
⬜ Only when using HCP Terraform.
```
**Question 7 (True/False):**

A failed `check` block assertion will prevent `terraform apply` from proceeding.

**Question 8 (Multiple Choice):**
Which function converts a Terraform map into a JSON string?
```
⬜ `jsondecode()`
⬜ `jsonencode()`
⬜ `yamldecode()`
⬜ `tostring()`
```
**Question 9 (Multiple Answer):**

Which statements about the `sensitive = true` flag are correct? (Select TWO)
```
⬜ It encrypts the value in the state file.
⬜ It redacts the value from CLI output.
⬜ It prevents the value from being used in resource arguments.
⬜ It propagates to any expressions that reference the sensitive variable.
```
**Question 10 (True/False):**

The `replace_triggered_by` lifecycle rule can reference resources, data sources, or specific resource attributes.

**Question 11 (Multiple Choice):**

You need to wait 30 seconds after creating an `aws_instance` before creating an `aws_route53_record`. How should you implement this?
```
⬜ Use `depends_on` with a `time_sleep` resource.
⬜ Add `sleep 30` to the `user_data` script.
⬜ Use the `create_before_destroy` lifecycle rule.
⬜ Terraform automatically handles API propagation delays.
```
**Question 12 (Multiple Choice):**

What is the result of the expression `merge({a=1}, {b=2})`?
```
⬜ `{a=1, b=2}`
⬜ `[{a=1}, {b=2}]`
⬜ `["a", "b"]`
⬜ `[1, 2]`
```
**Question 13 (True/False):**

Postconditions are evaluated after a resource is created or updated, and a failed postcondition will roll back the resource creation.

**Question 14 (Multiple Choice):**

Which function would you use to find all elements in a list that match a condition?
```
⬜ `contains()`
⬜ A `for` expression with an `if` clause
⬜ `filter()`
⬜ `lookup()`
```
**Question 15 (Multiple Answer):**

Which of the following are valid arguments to the `templatefile()` function? (Select TWO)
```
⬜ The path to the template file.
⬜ A list of variables to interpolate.
⬜ A map of variable assignments.
⬜ A boolean flag for escaping.
```
**Question 16 (True/False):**

The `terraform console` command allows you to test expressions and functions interactively against the current state.

**Question 17 (Multiple Choice):**

You want to ignore manual changes to the `tags` attribute of an AWS instance so Terraform doesn't revert them. Which lifecycle rule should you use?
```
⬜ `prevent_destroy`
⬜ `create_before_destroy`
⬜ `ignore_changes`
⬜ `replace_triggered_by`
```
**Question 18 (Multiple Answer):**

Which of the following are valid ways to validate input variables? (Select TWO)
```
⬜ `validation` block within the `variable` block.
⬜ `precondition` block within the `variable` block.
⬜ Using the `sensitive = true` flag.
⬜ Setting a restrictive `type` constraint.
```
**Question 19 (True/False):**

Using the Vault provider to generate dynamic AWS credentials eliminates the need to store long-lived AWS credentials in Terraform state.

**Question 20 (Multiple Choice):**

What does the `cidrsubnet("10.0.0.0/16", 8, 2)` function return?
```
⬜ `10.0.0.2/24`
⬜ `10.0.2.0/24`
⬜ `10.2.0.0/16`
⬜ `10.0.0.2/32`
```

<details>

<summary>Answers</summary>

**Answers & brief explanations**

1. True — The `for` expression iterates over the list and applies the `upper()` function to each element, resulting in `["A", "B"]`.
2. `templatefile()` is the correct function for rendering a template file with variable substitution.
3. `create_before_destroy`, `ignore_changes`, and `replace_triggered_by` are valid lifecycle rules. `prevent_apply` and `timeout` are not valid lifecycle rules.
4. True — `depends_on` can be used in a `module` block to ensure that the module's resources are created after another module's resources.
5. Both B and C — You can access the value using `var.tags["env"]` or `lookup(var.tags, "env")`.
6. Before the resource is created — Preconditions are evaluated before creating or updating a resource.
7. False — A failed `check` block assertion produces a warning but does not prevent `terraform apply` from proceeding.
8. `jsonencode()` converts a Terraform map into a JSON string.
9. It redacts the value from CLI output, and it propagates to any expressions that reference the sensitive variable. The `sensitive` flag does not encrypt the value in the state file, nor does it prevent usage in resource arguments.
10. True — The `replace_triggered_by` lifecycle rule can reference resources, data sources, or specific resource attributes to trigger replacement when they change.
11. Use `depends_on` with a `time_sleep` resource — This is a common pattern to handle API propagation delays.
12. `{a=1, b=2}` — The `merge()` function combines the two maps into one.
13. True — Postconditions are evaluated after a resource is created or updated, and if a postcondition fails, Terraform will roll back the resource creation or update.
14. A `for` expression with an `if` clause — Terraform does not have a built-in `filter()` function, but you can achieve filtering with a `for` expression.
15. The path to the template file, and a map of variable assignments — These are the valid arguments for `templatefile()`.
16. True — The `terraform console` command allows you to test expressions and functions interactively against the current state.
17. `ignore_changes` — This lifecycle rule tells Terraform to ignore changes to the specified attribute, allowing manual updates without Terraform reverting them.
18. `validation` block within the `variable` block, and setting a restrictive `type` constraint — These are valid ways to validate input variables. `precondition` blocks are not used within `variable` blocks, and `sensitive = true` is for redaction, not validation.
19. True — Using the Vault provider to generate dynamic AWS credentials eliminates the need to store long-lived AWS credentials in Terraform state, as the credentials are generated dynamically and have a short TTL.
20. `10.0.2.0/24` — The `cidrsubnet()` function creates a subnet within a given CIDR block, and the third argument specifies the subnet number.

</details>


Wow that was a lot of information! You should now have a solid understanding of Terraform's configuration language, including expressions, functions, dependencies, validation, and sensitive data management. This knowledge will be crucial for passing the Terraform Associate exam and for writing effective Terraform configurations in real-world projects.

Lets move on to [Objective 5: Terraform Modules](./05-Terraform-modules.md) where we will explore the concept of modules, how to create and use them, and best practices for module development.