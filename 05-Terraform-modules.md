# Objective 5: Terraform Modules

This objective covers how to use modules in Terraform, including how Terraform sources modules, variable scope within modules, using modules in configuration, and managing module versions. you will come to appreciate how modules enable reusability, abstraction, and organization in your Terraform configurations. By the end of this section, you should be able to confidently use modules to structure your infrastructure as code and understand how to manage their lifecycle effectively.

> Note it is mostly used for larger projects, but it is important to understand the concept of modules and how to use them effectively. Even if you are working on a small project, using modules can help you organize your code and make it more maintainable in the long run.

## Objective

- [Objective 5a: Explain how Terraform sources modules](#objective-5a-explain-how-terraform-sources-modules)

- [Objective 5b: Describe variable scope within modules](#objective-5b-describe-variable-scope-within-modules)

- [Objective 5c: Use modules in configuration](#objective-5c-use-modules-in-configuration)

- [Objective 5d: Manage module versions](#objective-5d-manage-module-versions)

- [Objective 5: Quiz](#objective-5-quiz)



## Objective 5a: Explain how Terraform sources modules

*Source: [`Modules overview`](https://developer.hashicorp.com/terraform/tutorials/modules/module), [`Use registry modules in your configuration`](https://developer.hashicorp.com/terraform/tutorials/modules/module-use), [`Find and use modules`](https://developer.hashicorp.com/terraform/registry/modules/use), [`Module Composition`](https://developer.hashicorp.com/terraform/tutorials/modules/module-composition), [`Standard Module Structure`](https://developer.hashicorp.com/terraform/tutorials/modules/module-structure)*

### 5a.1: What is a Module?

A module is a container for a group of related Terraform resources that are managed together. Every Terraform configuration has at least one module: the root module, which consists of the `.tf` files in your main working directory.

When you explicitly call a module using a `module` block, you create a **child module**. A child module can itself call other modules, forming a hierarchy.

**Why modules?**

- **Reusability**: Write a standard "web server" setup once, use it dozens of times.

- **Abstraction**: Hide complexity behind a simple interface (inputs and outputs).

- **Organization**: Group logically related resources together.

- **Collaboration**: Share modules via the Terraform Registry or internal registries.

### 5a.2: The `module` Block

You invoke a module using the module block:

```hcl
module "web_server" {
  source = "./modules/web_server"

  instance_type = "t3.micro"
  ami_id        = "ami-abc123"
}
```

If the module comes from the Terraform Registry, you would add a `version` argument as well.

Explanation of the block:

- `module "web_server" {`: This declares a child module. web_server is the local name you'll use to reference this module's outputs (e.g., `module.web_server.instance_id`).

- `source = "./modules/web_server"`: This is the source address. It tells Terraform where the module's code lives. The format depends on the type of source (see next section).

- `version = "1.0.0"`: (Optional but recommended) Specifies which version of the module to use. Only valid for modules hosted in registries.

- `instance_type = "t3.micro"`: This is an input variable passed to the child module. The child module must declare a `variable "instance_type" {}` block to accept this value.

- `ami_id = "ami-abc123"`: Another input variable passed to the module

### 5a.3: Module Sources

Terraform can fetch modules from many locations. The `source` argument determines the type and location.

Source Type|Syntax Example|Notes
-|-|-
Local path|`"./modules/network"`|A relative or absolute path on the local filesystem. No `version` argument needed. Fastest for development.
Terraform Registry|`"hashicorp/consul/aws"`|Public or private registry. Supports `version` constraint.
GitHub|`"github.com/org/repo"` or `"git::https://github.com/org/repo.git"`|Clones the Git repository. Can specify `ref` for branch/tag/commit.
Bitbucket|`"bitbucket.org/org/repo"`|Similar to GitHub.
Generic Git|`"git::https://example.com/repo.git"`|Any Git repository.
S3 bucket|`"s3::https://s3.amazonaws.com/my-bucket/module.zip"`|Downloads and unzips an archive.
HTTP URLs|`"https://example.com/module.zip"`|Downloads an archive.

Registry modules (shorthand):

- `hashicorp/consul/aws` → points to the `aws` submodule in the `consul` module published by `hashicorp`.

- `hashicorp/consul/aws//modules/consul-cluster` — the `//` separates the module path from a subdirectory inside the repository.

Local modules:

- `./modules/network` — looks for the module in the relative directory modules/network. No version negotiation; changes are immediate.

Git modules:

- `github.com/org/repo?ref=v1.2.0` — clone the repo and checkout the tag `v1.2.0`.

- `github.com/org/repo?ref=feature-branch` — uses a branch.

> **Important for exam:** When you use a local path, Terraform reads the module source directly from that directory on disk. There is no registry download step, so changes to the local module are reflected the next time you run a plan or apply.

When you use a remote source (registry, Git, S3), Terraform downloads the module into `.terraform/modules/` during `terraform init`. The module files are cached there. To update a remote module to a newer version, you must run `terraform init -upgrade` or `terraform get -update`.

### 5a.4: Module Installation

During `terraform init`, Terraform:

1. Reads all `module` blocks.

2. Resolves the `source` and `version` for each remote module.

3. Downloads remote modules into `.terraform/modules/MODULE_NAME/`.

4. Reuses cached modules when the resolved version matches.

To force an update, use `terraform init -upgrade` (for providers and registry modules) or `terraform get -update` (for all modules).

### 5a.5: Standard Module Structure

The official study guide recommends a specific file layout for reusable modules:

```text
my-module/
├── main.tf          # Primary resource definitions
├── variables.tf     # Input variable declarations
├── outputs.tf       # Output value declarations
├── README.md        # Documentation
├── LICENSE           # License file
├── modules/         # Nested submodules (optional)
│   └── submodule/
└── examples/        # Usage examples (optional)
    └── basic/
```

Explanation of standard files:

- `main.tf` — The main entry point. Contains `resource` and `data` blocks. For simple modules, this is the only file needed.

- `variables.tf` — Contains `variable` blocks that define the module's input interface. All variables should have description strings.

- `outputs.tf` — Contains `output` blocks that expose the module's results to the parent module. All outputs should have `description` strings.

- `README.md` — Markdown documentation. The Terraform Registry displays this. Should describe what the module does, its inputs, outputs, and provide an example.

- `LICENSE` — The license under which the module is distributed. Important for public modules; organizations often require a clear license before adopting a module.

- `modules/` — Contains nested submodules. These are for internal organization; they can be called by the root module of the module. External users can also call them directly using the `//` subdirectory syntax.

- `examples/` — Contains example configurations showing how to use the module. Not used by Terraform itself, but helpful for documentation.

---

### 5a.6: Hands-On Lab — Sourcing a Local Module

We'll create a local module and use it.

**Directory structure:**

```text
obj5a-lab/
├── main.tf
└── modules/
    └── greeting/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

#### Step 1: Create the child module `modules/greeting/main.tf`

```hcl
# This resource creates a local file with a personalized greeting.
# The content of the file is constructed from two input variables:
# 'recipient' (the person to greet) and 'salutation' (the greeting word).

resource "local_file" "greeting_file" {
  # The filename is fixed; we use path.module to ensure it's relative to the module directory.
  filename = "${path.module}/greeting.txt"
  # The content combines the salutation and recipient, separated by a comma.
  content  = "${var.salutation}, ${var.recipient}!"
}
```

Explanation of the resource block:

- `resource "local_file" "greeting_file" {` — Declares a managed resource of type `local_file` with local name `greeting_file`. Terraform will manage this file's lifecycle.

- `filename = "${path.module}/greeting.txt"` — Sets the absolute path to the output file. `path.module` is a built-in value that equals the filesystem path of the module where this expression is evaluated (here, ./modules/greeting). So the file will be ./modules/greeting/greeting.txt.

- `content = "${var.salutation}, ${var.recipient}!"` — Sets the file content. It uses string interpolation to insert the values of two input variables, followed by an exclamation mark.

#### Step 2: Create `modules/greeting/variables.tf`

```hcl
# Declares the input variables for the greeting module.
# Users of this module must provide a value for 'recipient'; 'salutation' has a default.

variable "recipient" {
  type        = string
  description = "The name of the person to greet."
  # No default; this makes the variable required.
}

variable "salutation" {
  type        = string
  description = "The greeting word, e.g., 'Hello'."
  default     = "Hello"  # If not provided, defaults to "Hello".
}
```

Explanation of the variable blocks:

- `variable "recipient" {` — Defines an input variable named recipient.

- `type = string` — Constrains the variable to accept only string values.

- `description = "..."` — Documents the purpose of the variable (used by the Registry and IDE integrations).

- `# No default; this makes the variable required.` — Because no `default` is set, the user must provide a value for recipient when calling the module.

- `variable "salutation" {` — Defines a second input variable.

- `default = "Hello"` — Provides a fallback value. The user may omit this argument, in which case "Hello" is used.

#### Step 3: Create `modules/greeting/outputs.tf`

```hcl
# Exposes the path of the created file to the parent module.

output "file_path" {
  value       = local_file.greeting_file.filename
  description = "The path to the greeting file."
}
```

Explanation of the output block:

- `output "file_path" {` — Declares an output value named `file_path`.

- `value = local_file.greeting_file.filename` — The value to export. It references the `filename` attribute of the `local_file.greeting_file` resource we created in `main.tf`.

- `description` — Documents the output.

#### Step 4: Create the root module `main.tf`

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

# Call the child module from the local filesystem.

module "my_greeting" {
  source = "./modules/greeting"  # The relative path to the module directory

  # Pass a value for the required 'recipient' variable
  recipient = "Terraform Student"
  # 'salutation' is optional; we override the default "Hello" with "Welcome"
  salutation = "Welcome"
}

# Output the path of the file created inside the module, accessible via module.<NAME>.<OUTPUT>
output "greeting_file_path" {
  value = module.my_greeting.file_path
}
```

Explanation of the root module:

- `terraform { ... }` — Sets up the required provider (`local`) with a version constraint.

- `module "my_greeting" {` — Creates a child module instance named `my_greeting`.

- `source = "./modules/greeting"` — Tells Terraform the module code lives in the ./modules/greeting directory relative to the root module.

- `recipient = "Terraform Student"` — Sets the `recipient` input variable of the child module to the string "Terraform Student".

- `salutation = "Welcome"` — Overrides the default `salutation` value; replaces "Hello" with "Welcome".

- `output "greeting_file_path" {` — Creates an output in the root module that will display the path of the file created by the child module.

- `value = module.my_greeting.file_path` — Uses the `module.MODULE_NAME.OUTPUT_NAME` syntax to reference the child module's output `file_path`.

#### Step 5: Run the workflow

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

Observe that the file `modules/greeting/greeting.txt` is created with content "Welcome, Terraform Student!".

---

### 5a.7: Module design patterns and best practices

Before you publish or reuse a module, choose a clear role for it. Modules generally fall into two patterns:

- **Leaf modules (resource modules):** Encapsulate a small, well-scoped set of resources (for example, a single database, an S3-backed static website). These are low-level building blocks intended to be composed.
- **Composition modules (service modules):** Combine multiple leaf modules into an opinionated higher-level component (for example, `web-service` that wires load balancers, autoscaling groups, and security groups together). These are what application teams consume directly.

#### Design guidance:

- Prefer small, focused leaf modules with simple, well-documented inputs and outputs. This keeps reuse flexible.
- Avoid exposing internal resource names or implementation-specific attributes as outputs; instead export stable logical values (IDs, endpoints, connection strings) that callers need.
- Keep module interfaces stable. Bump versions with semantic versioning and document breaking changes in the `CHANGELOG`.
- Provide `examples/` that show minimal and recommended usage (one example for simple usage, one for production-like usage).
- Use `required_providers` in the module's `terraform` block to document provider requirements, but do not hard-code provider configuration inside child modules — let the root module provide the provider configs and use `providers` mappings when needed.

#### Testing and publishing:

- Validate modules with `terraform validate` and `terraform plan` in your example folders. For unit-style tests, use Terratest (Go) or kitchen-terraform; for lightweight checks, run `terraform fmt` and `tflint` in CI.
- Tag releases for registry or git-hosted modules. Use tags (vX.Y.Z) so callers can pin modules to stable references.
- Document inputs, outputs, and permissions (IAM roles the module requires) clearly in the `README.md`.

Now a short quiz to check understanding.

---

### 5a.8: Mini-Quiz - Module Sources

1. True/False: When a module is sourced from a local path (e.g., `"./modules/network"`), you must run `terraform get` to download it before use.

2. Multiple Choice: Which of the following is a valid module source for a module hosted on the public Terraform Registry?

```
A. "registry.terraform.io/hashicorp/consul/aws"
B. "hashicorp/consul/aws"
C. "aws/consul/hashicorp"
D. "consul/aws"
```

3. Multiple Answer: Which files are part of the standard module structure? **(Select FOUR)**

```
A. main.tf
B. variables.tf
C. terraform.tfstate
D. outputs.tf
E. README.md
```

<details>

<summary>Show answers</summary>

**Answers & brief explanations**

1. **False.** Local modules are loaded from the filesystem, so there is nothing to download.

2. **B.** The shorthand registry format is `NAMESPACE/MODULE_NAME/PROVIDER`. The longer form would be `registry.terraform.io/hashicorp/consul/aws`, but the registry hostname can usually be omitted.

3. **A, B, D, E.** The standard reusable module layout includes `main.tf`, `variables.tf`, `outputs.tf`, and `README.md`. `terraform.tfstate` belongs to the root working directory, not the module package.

> `README.md` is not required for Terraform to execute the module, but it is the file that makes the module understandable to other humans. Think of it as the difference between "this works" and "this works, and here's how not to set it on fire."



</details>

## Objective 5b: Describe variable scope within modules

*Source: [`Manage values in modules`](https://developer.hashicorp.com/terraform/language/v1.12.x/values), [`Build and use a Local Module`](https://developer.hashicorp.com/terraform/tutorials/modules/module-create), [`Module Block Reference`](https://developer.hashicorp.com/terraform/language/v1.12.x/block/module)*

### 5b.1: Variable Scoping

Each Terraform module has its own **namespace**. Variables defined in a child module are local to that module. They cannot be directly accessed by the parent or sibling modules.

The only way to pass data between modules is:

- **Parent → Child:** Via input variables (arguments in the `module` block).

- **Child → Parent:** Via output values (referenced as `module.NAME.OUTPUT`).

No direct sibling communication. To pass data between two sibling modules, you must route it through the root module: 
> Module A outputs a value → Root module captures it → Root module passes it as an input to Module B.

```hcl
module "network" {
  source = "./modules/network"
}
module "compute" {
  source = "./modules/compute"
  vpc_id = module.network.vpc_id   # Root routes data from network to compute
}
```

### 5b.2: Variable Input Interface

When you call a child module, the argument values in the `module` block are matched to the `variable` declarations inside the child module by name.

```hcl
# Inside child module
variable "instance_type" { ... }
variable "instance_count" { ... }
```

```hcl
# In parent
module "web" {
  source = "./modules/web"
  instance_type = "t3.micro"    # Maps to variable "instance_type"
  instance_count = 3            # Maps to variable "instance_count"
}
```

**Requirements:**

- Variables without a `default` are required. The parent must provide them.

- Variables with a `default` are optional. The parent may omit them.

- The types must match. Terraform will attempt automatic conversion.

- Variables marked `sensitive` will have their values redacted in plan output, but the parent can still pass them.

### 5b.3: Output Values

A module's outputs are accessed by the parent using `module.MODULE_NAME.OUTPUT_NAME`.

```hcl
output "instance_ids" {
  value = module.web.instance_ids
}
```

**Key distinction:** Only the root module outputs are displayed in the CLI after `apply`. Child module outputs are not displayed unless explicitly output by the root module.

### 5b.4: Provider Scoping

By default, a child module inherits the default (non-aliased) provider configurations from its parent. It does not need to declare a `provider` block.

If a child module needs a different provider configuration (e.g., different region or different account), the parent must pass it explicitly using the `providers` argument in the `module` block:

```hcl
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

module "app_west" {
  source = "./modules/app"
  providers = {
    aws = aws.west    # The child's default 'aws' provider is mapped to this aliased config
  }
}
```

We will cover this in depth in Objective 5c.

Deeper example — child declares provider requirement, parent supplies configurations:

In the child module (`modules/app/terraform {}`):

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0"
    }
  }
}
```

Child resources reference the `aws` provider normally:

```hcl
resource "aws_s3_bucket" "assets" {
  bucket = var.bucket_name
}
```

In the parent module, provide one or more `aws` configurations and map them if needed:

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "replica"
  region = "us-west-2"
}

module "primary_app" {
  source = "./modules/app"
  # uses default aws provider
}

module "replica_app" {
  source = "./modules/app"
  providers = {
    aws = aws.replica  # map the child's 'aws' provider to the aliased config
  }
}
```

**Important notes:**

- The child module should declare `required_providers` to document the providers it needs. The actual provider configuration (credentials, region, alias) is supplied by the parent.
- The `providers` map in a `module` block maps provider names used in the child to provider configuration objects from the parent. This is how you run the same module in multiple regions/accounts.

---

### 5b.5: Hands-On Lab — Variable Scoping

Goal: Demonstrate that sibling modules cannot directly access each other's variables.

Directory structure:

```text
obj5b-lab/
├── main.tf
├── modules/
│   ├── producer/
│   │   ├── main.tf
│   │   └── outputs.tf
│   └── consumer/
│       ├── variables.tf
│       └── main.tf
```

**Producer module (`modules/producer/main.tf`):**

```hcl
# This module generates a random pet name and exposes it as an output.
resource "random_pet" "name" {
  length = 2
}
```

**Producer module (`modules/producer/outputs.tf`):**

```hcl
# The generated pet name is exported so the parent module can use it.
output "pet_name" {
  value = random_pet.name.id
}
```

**Consumer module (`modules/consumer/variables.tf`):**

```hcl
# This module expects a name to be passed in from the parent.
variable "base_name" {
  type = string
  description = "A name prefix for the file."
}
```

**Consumer module (`modules/consumer/main.tf`):**

```hcl
# Creates a file whose name is based on the input variable.
resource "local_file" "record" {
  filename = "${path.module}/${var.base_name}.txt"
  content  = "Record for ${var.base_name}"
}
```

**Root module (`main.tf`):**

```hcl
terraform {
  required_providers {
    random = { source = "hashicorp/random", version = "~> 3.6" }
    local  = { source = "hashicorp/local", version = "~> 2.5" }
  }
}

module "producer" {
  source = "./modules/producer"
  # No input variables needed; random_pet requires no inputs.
}

module "consumer" {
  source    = "./modules/consumer"
  # Pass the pet name from producer into the consumer.
  # This demonstrates parent routing data between sibling modules.
  base_name = module.producer.pet_name
}

# The consumer module's output could be exposed here, but we don't have one yet. We'll just apply.
```

#### Steps:

1. `terraform init`

2. `terraform apply -auto-approve`

3. Observe that the consumer module creates a file like `modules/consumer/<pet-name>.txt`.

4. If you attempt to reference `module.producer.pet_name` inside the consumer module (without passing it as a variable), Terraform would error — variables are scoped per module.

---

### 5b.6: Mini-Quiz - Variable Scope

1. True/False: A child module can directly access a sibling module's output values without the parent's intervention.

2. Multiple Choice: How does a child module receive data from its parent?

```
A. By reading the parent's output blocks directly.
B. Via input blocks in the child.
C. Through input variables declared in the child and passed as arguments in the module block.
D. By using the terraform_remote_state data source.
```

3. True/False: If a child module does not define any `provider` block, it cannot use any resources.

<details>
<summary>Show answers</summary>

**Answers & brief explanations**

1. **False.** Sibling modules cannot directly access each other's variables or outputs. Data must be passed through the parent module.
2. **C.** The child module defines input variables, and the parent module passes values to those variables in the `module` block.

3. **False.** A child module can still use resources because it inherits the parent's default provider configuration. The child only needs its own provider block when it must define a provider configuration that is different from the parent, and even then that configuration should usually live in the parent and be passed in with `providers`.

</details>


## Objective 5c: Use modules in configuration

*Source: [`Use registry modules in your configuration`](https://developer.hashicorp.com/terraform/tutorials/modules/module-use), [`Output Block Reference`](https://developer.hashicorp.com/terraform/language/v1.12.x/block/output), [`Module Composition`](https://developer.hashicorp.com/terraform/language/v1.12.x/modules/develop/composition), [`Manage values in modules`](https://developer.hashicorp.com/terraform/language/v1.12.x/values)*

### 5c.1: Calling a Module

The `module` block syntax:

```hcl
module "NAME" {
  source = "SOURCE"
  version = "VERSION"   # optional for registries

  INPUT_VARIABLE_NAME = VALUE
  ...
}
```

- `NAME`: Unique identifier within the parent module. Used to reference the module's outputs.

- `source`: Location of the module code (required).

- `version`: (Optional) Version constraint string for registry modules.

Input variable assignments: Key-value pairs mapping child variable names to values.

---

### 5c.2: Multi-Instance Modules: `count` and `for_each`

You can create multiple instances of a module using `count` or `for_each` (Terraform 0.13+).

**Using `count`:**

```hcl
module "server" {
  source   = "./modules/server"
  count    = 3
  hostname = "server-${count.index}"
}
```

This creates three instances of the module: `module.server[0]`, `module.server[1]`, `module.server[2]`.

**Using `for_each`:**

```hcl
module "server" {
  source   = "./modules/server"
  for_each = toset(["web", "db", "cache"])
  hostname = each.key
}
```

This creates `module.server["web"]`, `module.server["db"]`, `module.server["cache"]`.

**Accessing outputs from multi-instance modules:**

```hcl
output "all_ips" {
  value = [for s in module.server : s.private_ip]
}
```

### 5c.3: Explicit Dependencies (depends_on)

Module blocks support `depends_on` to ensure ordering when implicit references don't capture the dependency.

```hcl
module "database" {
  source = "./modules/db"
}

module "app" {
  source     = "./modules/app"
  depends_on = [module.database]
}
```

This ensures `module.database` is fully applied before `module.app` starts, even if no attributes are directly referenced.

---

### 5c.4: Providers and Modules

**Default inheritance:** Child modules implicitly inherit the default (non-aliased) provider configurations from the parent. They do not need to declare a `provider` block.

**Passing explicit providers:** Use the `providers` argument in the `module` block to map a child's provider name to a specific configuration from the parent.

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "replica"
  region = "us-west-2"
}

module "primary" {
  source = "./modules/app"
  # No providers argument; uses the default 'aws' provider.
}

module "replica" {
  source = "./modules/app"
  providers = {
    aws = aws.replica
  }
}
```

In the child module, resources using `aws` will use the corresponding provider passed by the parent.

**Important Note:** A child module should not contain a configured `provider` block (with credentials, region, etc.). It may include a `terraform { required_providers { ... } }` block to declare which providers and versions it needs, but actual provider configuration should live in the root (parent) module and be passed in via the `providers` map when needed.

Why configuring providers inside child modules is problematic:

- Provider configurations are global to an execution; when modules attempt to configure providers locally it can create ambiguous or conflicting configurations, especially when the module is instantiated multiple times with `count` or `for_each` or when different instances need different provider configurations. Keeping provider configuration in the root module preserves a single source of truth and makes multi-instance and aliased-provider scenarios predictable.

---

### 5c.5: Hands-On Lab — Using a Module with Multiple Instances

We'll create a module that generates a custom-named file, then use it multiple times with `for_each`.

Module structure:

```text
obj5c-lab/
├── main.tf
└── modules/
    └── named_file/
        ├── variables.tf
        └── main.tf
```

**Module `variables.tf` (`modules/named_file/variables.tf`):**

```hcl
variable "file_name" {
  type        = string
  description = "The name of the file to create (without extension)."
}
variable "suffix" {
  type        = string
  description = "A suffix to append to the filename."
}
```

**Module `main.tf` (`modules/named_file/main.tf`):**

```hcl
resource "local_file" "this" {
  filename = "${path.module}/${var.file_name}_${var.suffix}.txt"
  content  = "File created by module instance for ${var.file_name}."
}
```

**Root (`main.tf`):**

```hcl
terraform {
  required_providers {
    local = { source = "hashicorp/local", version = "~> 2.5" }
  }
}

locals {
  common_suffix = "v2"
}

module "documents" {
  source   = "./modules/named_file"
  for_each = toset(["alpha", "beta", "gamma"])

  file_name = each.key           # Passes the set element as file_name
  suffix    = local.common_suffix # Uses the local value for suffix
}

output "created_files" {
  value = { for k, m in module.documents : k => m.this.filename }
}
```

**Explanation of root `main.tf`:**

- `terraform { ... }` — Sets up the required `local` provider.

- `locals { common_suffix = "v2" }` — Defines a local value `common_suffix` that can be reused throughout this module.

- `module "documents" {` — Creates a child module instance (or set of instances) named `documents`.

- `source = "./modules/named_file"` — Points to the module directory.

- `for_each = toset(["alpha", "beta", "gamma"])` — Iterates over a set of three strings, creating three instances of the module. The key (each.key) is one of the strings.

- `file_name = each.key` — Passes the iteration key as the `file_name` input variable.

- `suffix = local.common_suffix` — Passes the local value `common_suffix` as the `suffix` input.

- `output "created_files" {` — Creates an output in the root module that shows all created filenames.

- `value = { for k, m in module.documents : k => m.this.filename }` — Uses a `for` expression to iterate over the module instances. For each instance (referenced by m), it retrieves the filename attribute of the local_file.this resource inside that module instance.

Run `terraform init && terraform apply -auto-approve` to see three files created.

---

### 5c.6: Hands-On Lab — Provider scoping and multi-region module instances

Goal: Run the same reusable module in two AWS regions by mapping provider configurations from the parent module into child modules.

Directory structure:

```text
obj5c-provider-lab/
├── main.tf
└── modules/
    └── bucket/
        ├── main.tf
        └── variables.tf
```

Child module (`modules/bucket/main.tf`):

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws" }
  }
}

variable "bucket_name" { type = string }

resource "aws_s3_bucket" "b" {
  bucket = var.bucket_name
  acl    = "private"
}

output "bucket_arn" {
  value = aws_s3_bucket.b.arn
}
```

Parent/root (`main.tf`):

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

module "east_bucket" {
  source = "./modules/bucket"
  bucket_name = "my-app-east-123"
  # uses default aws provider (us-east-1)
}

module "west_bucket" {
  source = "./modules/bucket"
  bucket_name = "my-app-west-123"
  providers = {
    aws = aws.west  # map child's 'aws' to the aliased parent provider
  }
}

output "east_arn" { value = module.east_bucket.bucket_arn }
output "west_arn" { value = module.west_bucket.bucket_arn }
```

Steps:

1. `terraform init`
2. `terraform plan` — verify two buckets will be created in different regions.
3. `terraform apply -auto-approve`
4. Confirm outputs show two different ARNs for each region.

**Important Notes:**

- The child declares `required_providers { aws = ... }` but does not configure credentials or region — the parent supplies those via provider blocks and the `providers` map.
- This pattern is how you run the same module across multiple accounts/regions safely.

---

### 5c.7: Mini-Quiz - Using Modules in Configuration

1. True/False: The `providers` argument inside a `module` block is used to map provider names from the child module to provider configurations in the parent.

2. Multiple Choice: You want to create three copies of a module, each with a different name. Which approach is most appropriate?

```
A. count = 3 with count.index to differentiate.
B. for_each = toset(["a","b","c"]) with each.key.
C. Copy-paste the module block three times.
D. Use a provider alias.
```

3. Multiple Answer: Which meta-arguments are available on a module block? (Select THREE)

```
A. depends_on
B. lifecycle
C. count
D. for_each
E. provider
```

<details>

<summary>Show answers</summary>

**Answers & brief explanations**

1. **True.** The `providers` argument allows you to specify which provider configuration the child module should use for its provider references.

2. **B.** Using `for_each` with a set of unique identifiers is the most appropriate way to create multiple instances of a module with different names.

3. **A, C, D.** The meta-arguments available on a module block include `depends_on`, `count`, and `for_each`. The `lifecycle` meta-argument is not valid on a module block, and `provider` is not a meta-argument; instead, you use the `providers` argument to specify provider configurations for the child module.

</details>


## Objective 5d: Manage module versions

*Source: [`Use registry modules in configuration`](https://developer.hashicorp.com/terraform/tutorials/modules/module-use), [`Module block reference: version argument`](https://developer.hashicorp.com/terraform/language/v1.12.x/block/module#version)*

### 5d.1: Version Constraints for Modules

When using modules from a registry (public or private), you can specify a `version` argument to control which version of the module is used.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"   # exact version
}
```

You can use all the same version constraint operators as for providers: `>=`, `<=`, `~>`, `=`, etc.

```hcl
version = "~> 5.0"   # any 5.x, not 6.0
version = ">= 4.0, < 5.0"
```

**Important Note:** If you omit `version`, Terraform will use the **latest** version available when the module is first downloaded. This can lead to unexpected changes when a new major version is released. 
> **Always pin module versions for stability.**

---

### 5d.2: Updating Modules

Modules are downloaded during `terraform init`. To update a module to a newer version:

1. Change the version constraint in the module block.

2. Run `terraform init -upgrade`.

The `-upgrade` flag tells Terraform to check the registry for newer versions and update `.terraform/modules/` accordingly.

Alternatively, `terraform get -update` also updates modules.

### 5d.3: Module Versioning and Git

For Git-sourced modules, versioning is handled by referring to a specific `ref` (branch, tag, or commit SHA).

```hcl
module "my_module" {
  source = "git::https://github.com/org/repo.git?ref=v1.2.0"
}
```

Here, `ref=v1.2.0` points to a Git tag. Tags are the recommended way to version Git-sourced modules.

No automatic update for Git sources. Terraform caches the clone. To update, you must change the `ref` and run `terraform init -upgrade` or `terraform get -update`.

### 5d.4: Module vs. Provider Versioning

There's an important difference:

- Provider versions are recorded in the dependency lock file (`.terraform.lock.hcl`) to ensure consistent versions across a team.

- Module versions are not recorded in the lock file. Consistency relies on the `version` argument in the `module` block itself. If you omit `version`, different team members could get different module versions.

**Best practice:** Always set exact or tightly-constrained versions for all registry modules in your configuration.

#### Module versioning and upgrade strategy (best practices):

- Use Semantic Versioning (SemVer) for published modules. Consumers should rely on `~>` constraints to allow non-breaking patch/minor updates (for example, `~> 1.2`), and pin major versions for breaking changes.
- Test module upgrades in an isolated environment (CI or a staging workspace) before applying to production. The recommended flow is: update the `version` (or `ref`) in a branch, run `terraform init -upgrade` in CI, run `terraform plan`, and review resource changes carefully.
- For Git-sourced modules, prefer tagging releases and referencing tags via `?ref=v1.2.3`. Avoid pointing to moving targets like `main` in production.
- Record the resolved module versions in your team's documentation or CI artifact. Unlike providers, module versions are not captured in `.terraform.lock.hcl`, so your team must enforce version discipline either through code review or CI checks.
- Consider using a module registry (private or public) that offers review, permissions, and release workflows for organizational modules.

#### Module testing and CI recommendations

Build a simple CI pipeline for module changes that runs these checks automatically:

- `terraform fmt` — ensures consistent formatting.
- `terraform validate` — catches syntax and basic semantic errors.
- `tflint` and `checkov` (optional) — catch policy/security and style issues.
- `terraform init` and `terraform plan` in the example directories — ensures the examples remain functional when provider versions change.
- Integration tests (optional): Use Terratest (Go) or kitchen-terraform to create/destroy real resources against a sandbox account; run these on merge to protected branches only.

Keep CI failures as blockers for merges that change module interfaces or provider requirements. Tag and release passing builds so consumers can pin to a known good version.

## Objective 5 Comprehensive Gauntlet (20 Questions)

### 5d.5: Hands-On Lab — Versioning with a Registry Module

Goal: Use the public `hashicorp/random` module (a fictional registry module for demonstration; we'll actually simulate by using a local module with version-like constraints to illustrate the concept).

Since we can't access a real registry without internet-calling a module, we'll modify the previous lab to demonstrate updating a module version. We'll use a local module but change the "version" by altering the source directory to simulate a new version.


**Demonstration steps (conceptual):**

1. In `main.tf`, set `version = "~> 1.0"` for a registry module.

2. Run `terraform init`. Terraform downloads the latest compatible 1.x release.

3. Later, change the constraint to `version = "~> 2.0"` and run `terraform init -upgrade`. Terraform refreshes the module to the newest compatible 2.x release.

This reinforces the exam rule: changing the version constraint alone is not enough for Terraform to fetch a newer registry module; you must run `terraform init -upgrade`.

---

### 5d.6: Mini-Quiz for 5d

1. True/False: Module versions are automatically recorded in `.terraform.lock.hcl` to ensure team consistency.

2. Multiple Choice: To upgrade a registry module to a new version, you should:

```
A. Modify the version constraint and run terraform apply.
B. Modify the version constraint and run terraform init -upgrade.
C. Delete the .terraform directory and run terraform apply.
D. Run terraform get -update only (no constraint change needed).
```

3. Multiple Answer: Which sources support a version argument? (Select TWO)
```
A. Local path ("./modules/network")
B. Terraform Registry ("hashicorp/consul/aws")
C. Git without ref ("git::https://example.com/repo.git")
D. Private Terraform Registry ("mycompany/module/provider")
```

<details>

<summary>Show answers</summary>

**Answers & brief explanations**

1. **False.** Module versions are not recorded in the lock file. Consistency relies on the `version` argument in the module block.

2. **B.** To upgrade a registry module, you must modify the version constraint and run `terraform init -upgrade` to fetch the new version.

3. **B, D.** The `version` argument is supported for modules sourced from the Terraform Registry, both public and private. Local paths and Git sources do not use `version`; Git sources are pinned with `ref` instead.

</details>


## Objective 5 Comprehensive Gauntlet (20 Questions)

Note down your answers, and check them against the provided answers at the end.

**Question 1 (True/False):**

A module's `output` block can directly reference a variable from a sibling module.

**Question 2 (Multiple Choice):**

Which `source` value correctly points to a local module in the directory `./modules/network`?
```
⬜ "./network"
⬜ "./modules/network"
⬜ "local/network"
⬜ "file://./modules/network"
```

**Question 3 (Multiple Answer):**

When sourcing a module from a Git repository, which of the following can be used to specify a particular version? (Select TWO)
```
⬜ version = "1.0.0" argument
⬜ ref=v1.0.0 query parameter in the source URL
⬜ branch=main query parameter
⬜ commit=abc123 query parameter
```

**Question 4 (True/False):**

The `providers` argument in a `module` block allows a child module to define its own provider configurations that the parent cannot override.

**Question 5 (Multiple Choice):**

How do you reference an output named `vpc_id` from a child module named `network`?
```
⬜ data.module.network.vpc_id
⬜ module.network.output.vpc_id
⬜ module.network.vpc_id
⬜ network.vpc_id
```

**Question 6 (Multiple Choice):**

What is the default behavior for provider configurations in child modules?
```
⬜ Child modules must always define their own provider blocks.
⬜ Child modules inherit only aliased providers from the parent.
⬜ Child modules inherit the default (non-aliased) provider configurations from the parent.
⬜ Child modules cannot use providers.
```

**Question 7 (True/False):**

A local module sourced via a relative path will be updated automatically when you run `terraform init -upgrade` if changes are made to the module directory.

**Question 8 (Multiple Answer):**

Which of the following are valid arguments in a `module` block? (Select THREE)
```
⬜ source
⬜ version
⬜ backend
⬜ depends_on
⬜ lifecycle
```

**Question 9 (True/False):**

A child module can directly read the root module's output values.

**Question 10 (Multiple Choice):**

You have a module that creates an AWS VPC. You need to create three identical VPCs for three different environments. Which approach should you use?
```
⬜ Use count = 3 inside the module's aws_vpc resource.
⬜ Use for_each on the module block with environment names.
⬜ Define three separate module blocks.
⬜ Use a provider alias for each environment.
```

**Question 11 (Multiple Choice):**

What happens if you omit the `version` argument in a module block that references a module from the public Terraform Registry?
```
⬜ Terraform will error, as version is required.
⬜ Terraform will use the latest version when the module is first downloaded.
⬜ Terraform will use the version pinned in .terraform.lock.hcl.
⬜ Terraform will prompt you to choose a version.
```

**Question 12 (True/False):**
A module's `README.md` is required for the module to be usable by Terraform.

**Question 13 (Multiple Answer):**
Which of the following statements about variable scope in modules are true? (Select TWO)
```
⬜ Variables defined in a child module can be directly accessed by the root module.
⬜ A child module's outputs can be accessed by the root module using `module.NAME.OUTPUT_NAME`.
⬜ Two sibling modules can communicate directly without routing through the root module.
⬜ A root module passes values to a child module via the module block's arguments.
```

**Question 14 (True/False):**

When using `for_each` on a module block, the module's resources are all created in parallel, ignoring any intra-module dependencies.

**Question 15 (Multiple Choice):**

Which file in the standard module structure contains the declarations for the module's input variables?
```
⬜ main.tf
⬜ outputs.tf
⬜ variables.tf
⬜ README.md
```

**Question 16 (Multiple Answer):**

Which of the following can be used to update a module that is sourced from the Terraform Registry? (Select TWO)
```
⬜ Run terraform apply -upgrade
⬜ Run terraform init -upgrade
⬜ Change the version constraint in the module block
⬜ Delete the module's outputs.tf file
```

**Question 17 (True/False):**
A child module can contain its own `terraform` block with `required_providers` to specify which providers and versions it needs.


**Question 18 (Multiple Choice):**

What is the purpose of the `//` in a module source like `"hashicorp/consul/aws//modules/consul-cluster"`?

```
⬜ It is a comment delimiter.
⬜ It separates the registry address from a subdirectory path within the module repository.
⬜ It indicates that the module uses multiple providers.
⬜ It is part of the version constraint syntax.
```

**Question 19 (True/False):**

The `depends_on` argument in a `module` block can reference individual resources inside that module.

**Question 20 (Multiple Choice):**

You want to share data from Module A to Module B. Module A outputs a value `subnet_ids`. How do you pass this to Module B?

```
⬜ In Module B's main.tf, reference module.A.subnet_ids directly.
⬜ In the root module, set an argument in Module B like subnet_ids = module.A.subnet_ids.
⬜ Use the terraform_remote_state data source in Module B pointing to Module A.
⬜ Define a global variable accessible by both modules.
```

<details>

<summary>Answer</summary>

**Answers & expanded explanations**

1. False — Module outputs cannot directly reach into a sibling module. Modules are isolated: to share data between siblings you must export an output from the producing module to the parent and then pass that value into the consuming module via a module argument.

2. B — `"./modules/network"` is the correct relative path form. While `file://` URLs can also point to local files, the conventional and portable way in Terraform code is the relative path.

3. B — For Git-sourced modules you pin using the `ref` query parameter in the source URL (`?ref=v1.0.0`), which can be a tag, branch name, or commit SHA. The `version` argument is only for registry-hosted modules.

4. False — The `providers` argument in a `module` block maps which parent provider configuration the child should use; it does not let the child hide or lock out the parent's ability to provide provider configs. In practice, provider configuration (credentials/region) should be supplied by the parent.

5. C — Use `module.network.vpc_id` to read the `vpc_id` output from the child module named `network`.

6. C — By default a child module inherits the root module's default (non-aliased) provider configuration. To use a different configuration (different region/account), the parent creates an aliased provider and maps it via the `providers` argument.

7. False — Local modules are read directly from disk each time Terraform runs; you do not need `terraform init -upgrade` to pick up edits to local source files. (Remote modules are cached in `.terraform/modules` and require an update action.)

8. A, B, D — `source`, `version` (registry only), and `depends_on` are valid on a `module` block. `backend` and `lifecycle` are not valid module-level arguments.

9. False — A child module cannot read root outputs directly; the root can read the child's outputs and re-export them if desired, or route values from one child into another.

10. B — Use `for_each` on the module block with a set/map of environment identifiers to create distinct module instances with stable instance keys. This is clearer and less error-prone than trying to mutate a single module with internal `count`.

11. B — Omitting `version` for a registry module causes Terraform to select the latest available at initial download — which can introduce drift for different users. Always pin versions for reproducibility.

12. False — `README.md` is not required by Terraform to execute a module, but it is essential for humans: document inputs, outputs, examples, and any IAM or permission needs.

13. B, D — True: a child's outputs are accessible via `module.NAME.OUTPUT_NAME`, and the parent passes values into a child via the module block's argument names. Variables inside a child are not directly visible to the parent.

14. False — `for_each` creates multiple module instances, but Terraform still respects the dependency graph, including references inside each module instance; intra-module dependencies are honored.

15. C — `variables.tf` is the conventional place to declare input variables for a module.

16. B, C — To upgrade a registry module you typically change the `version` constraint and run `terraform init -upgrade`. Changing the constraint without reinitializing won't fetch the new code until you run the upgrade/init step.

17. True — A child module may include a `terraform { required_providers { ... } }` block to declare which providers and versions it expects; however, actual provider configuration (credentials, region, aliases) should be supplied by the parent to avoid conflicts and support multi-instance usage.

18. B — The `//` in a module source separates the module registry path from a subdirectory path inside the repository (useful for monorepos or modules with nested submodules).

19. False — `depends_on` on a module block may reference resources or modules in the parent module only. You cannot use a parent-level `depends_on` to reference internal resources inside the child module.

20. B — The normal pattern is: Module A exports `subnet_ids` via an output; in the root module you pass `subnet_ids = module.A.subnet_ids` into Module B's input variable when calling Module B.

</details>

## Ready for Objective 6

You’ve got the module model now: source it, scope it, compose it, and version it with intent. Next up is Objective 6, where state management turns Terraform from "nice config files" into a reliable system.

> Why did the Terraform module go to therapy? It had too many unresolved dependencies.

Congratulations, now let’s proceed to [Objective 6: Terraform State Management](./06-Terraform-state-management.md), where we’ll cover how Terraform tracks resources, manages state files, and handles remote backends for collaboration. this is where the magic of Terraform's infrastructure as code really comes alive, so buckle up! State management is the heart of Terraform's ability to manage infrastructure reliably and predictably.
