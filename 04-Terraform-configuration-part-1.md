# Objective 4: Terraform configuration

**Objective 4** is the largest and most detailed section. It covers the entire Terraform configuration language: resources, data sources, variables, outputs, expressions, functions, dependencies, conditions, and sensitive data management. This is where you learn to write Terraform code.

> Core mental model: Terraform does not run top to bottom like a script. It builds a dependency graph from references, validates inputs and types, then plans the smallest set of changes needed to make real infrastructure match the desired configuration.

> Given the volume, we will break this into two parts to ensure depth without overload.

## Objective 4 Part 1: 

Feel free to review the full content of Objective 4 Part 1, but here’s a quick outline of what we will cover, you are welcome to jump to any section directly:

- [Objective 4a: Use and differentiate resource and data blocks](#objective-4a-use-and-differentiate-resource-and-data-blocks)

- [Objective 4b: Refer to resource attributes and create cross-resource references](#objective-4b-refer-to-resource-attributes-and-create-cross-resource-references)

- [Objective 4c: Use variables and outputs](#objective-4c-use-variables-and-outputs)

- [Objective 4d: Understand and use complex types](#objective-4d-understand-and-use-complex-types)

- [Objective 4: Quiz](#objective-4-part-1-quiz)

- [Objective 4: Part 2](./04-Terraform-configuration-part-2.md)


## Objective 4a: Use and differentiate resource and data blocks

*Source: [`Resources`](https://developer.hashicorp.com/terraform/language/v1.12.x/resources), [`Query data sources`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/data-sources), [`Data Sources`](https://developer.hashicorp.com/terraform/language/v1.12.x/data-sources)*

### 4a.1: The `resource` Block — Managed Infrastructure

A `resource` block declares a piece of infrastructure that Terraform will manage. Management means Terraform will create, read, update, and delete (CRUD) this object to match the desired state in your configuration. In practice, a resource says, "Terraform owns this object and must keep it aligned with the code."

Syntax:

```hcl
resource "TYPE" "LOCAL_NAME" {
  # Provider-specific arguments
  argument1 = "value"
  argument2 = 123

  # Optional nested blocks
  nested_block {
    setting = true
  }
}
```

`TYPE:` The resource type defined by the provider. Format: provider_resource. Example: `aws_instance`, `azurerm_virtual_machine`, `local_file`.

`LOCAL_NAME:` A name you choose to reference this resource within the module. Must be unique per resource type in the module. This name has no effect on the actual cloud resource name.

**Resource Address:** The combination `TYPE.LOCAL_NAME` (e.g., `aws_instance.web`) uniquely identifies the resource in your configuration and state. When indexes are involved, the address also includes the instance key or index, which is why state and CLI commands can target one specific object.

**Meta-Arguments:**
Terraform provides built-in arguments that work on any resource type:

- `count:` Creates multiple instances of a resource.

- `for_each:` Creates multiple instances based on a map or set.

- `depends_on:` Explicitly defines dependencies.

- `provider:` Selects a non-default provider configuration (aliased).

- `lifecycle:` Controls resource lifecycle behavior (`create_before_destroy`, `prevent_destroy`, `ignore_changes`).

These are covered in depth in later objectives.

**Resource Behavior:**

- When you add a new `resource` block, Terraform plans to create it.

- When you modify arguments, Terraform plans to update in-place or replace (destroy + create) based on what the provider supports.

- When you remove a `resource` block, Terraform plans to destroy the real-world object.

### 4a.2: The `data` Block — Querying Existing Information

A `data` block (data source) instructs Terraform to read information from an external source without managing it. Data sources are read-only, so they are for discovery, lookup, and reuse - not ownership.

Syntax:

```hcl
data "TYPE" "LOCAL_NAME" {
  # Query parameters
  filter {
    name   = "tag:Name"
    values = ["my-server"]
  }
}
```

- `TYPE:` The data source type, defined by the provider. Example: aws_ami, aws_vpc, local_file.

- `LOCAL_NAME:` A local identifier for referencing the fetched data.

**Common Use Cases:**

- Fetch the latest AMI ID for an EC2 instance (so you don't hardcode AMIs).

- Look up a VPC ID by name.

- Read a file's contents locally.

Access outputs from another Terraform workspace (via `terraform_remote_state`).

The practical difference is that a resource changes the remote object if needed, while a data source only asks, "what already exists right now?"

**Data Source Behavior:**

- Data sources are read during `plan` if their arguments are known.

- If a data source argument depends on a managed resource that hasn't been created yet, the read is deferred to `apply`.

- Data sources do not create or modify anything. They only fetch information.

**Important Distinction:**

Aspect|`resource`|`data`
-|-|-
Purpose|Manage infrastructure|Query existing information
State|Full lifecycle in state file|Result cached in state, but not "managed"
Apply|Creates/Updates/Destroys|Only reads
ID format|`resource.TYPE.NAME`|`data.TYPE.NAME`

### 4a.3: Meta-Arguments for `data` Blocks

Data sources support the same meta-arguments as resources:

- `count` / `for_each`: Fetch multiple similar items.

- `depends_on`: Ensure the read happens after a specific resource is created.

- `provider`: Use an aliased provider configuration.

- `lifecycle`: Only `precondition` and `postcondition` are supported (no `create_before_destroy` etc.).

### 4a.4: Hands-On Lab — Resource vs. Data Source

Goal: Create a file using a `resource`, then read its content back using a `data` source.

Directory: `obj4a-lab`

File: `main.tf`

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

# Managed resource: we control this file's lifecycle
resource "local_file" "greeting" {
  content  = "Hello, Terraform!"
  filename = "${path.module}/greeting.txt"
}

# Data source: read the file we just created (and any existing one)
data "local_file" "read_greeting" {
  filename = local_file.greeting.filename
}

output "file_content" {
  value = data.local_file.read_greeting.content
}
```

**Steps:**

1. `terraform init` — Initializes the provider.

2. `terraform plan` — Observe that Terraform plans to create the file and then read it. The data source depends on the resource because of the reference local_file.greeting.filename.

3. `terraform apply` — File created, content output.

4. Manually edit `greeting.txt` to change text to `"Modified outside Terraform"`.

5. `terraform plan` — Terraform will detect that the file content differs from the state of the resource, and propose to update the resource back to `"Hello, Terraform!"`. The data source will re-read the file after the update.

6. `terraform apply` — File restored.

This demonstrates the fundamental difference: Terraform manages the resource, but only reads the data source.

### 4a.5: Mini-Quiz for 4a

1. True/False: A `data` block can be used to create a new resource if it doesn't already exist.

2. Multiple Choice: Which meta-argument is valid for a `data` block?

```
A. lifecycle { create_before_destroy = true }
B. depends_on
C. provisioner "local-exec"
D. prevent_destroy
```

3. True/False: The local name of a resource (`resource "aws_instance" "web"`) determines the Name tag of the EC2 instance in AWS.

<details>
<summary>Show answers</summary>

**Answers & brief explanations**

1. **FALSE** — A `data` block only reads existing resources. It cannot create a new resource if it doesn't exist.

2. **B** — `depends_on` is a valid meta‑argument for a data block. `create_before_destroy`, `prevent_destroy`, and `provisioner` are not supported on data sources.

3. **FALSE** — The local name `("web")` is a Terraform‑internal identifier, not the AWS Name tag. The Name tag must be set explicitly in the `tags` argument.

</details>


## Objective 4b: Refer to resource attributes and create cross-resource references

*Source: [`References to Named Values`](https://developer.hashicorp.com/terraform/language/v1.12.x/expressions/references), [`Resource Address Reference`](https://developer.hashicorp.com/terraform/cli/v1.12.x/state/resource-addressing), [`Create resource dependencies`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/dependencies)*

### 4b.1: Referencing Resource Attributes

Once you define a resource or data source, you can use its attributes elsewhere in the configuration. 

The syntax is:

- `RESOURCE_TYPE.LOCAL_NAME.ATTRIBUTE`

- `data.DATA_TYPE.LOCAL_NAME.ATTRIBUTE`

Example:

```hcl
resource "random_pet" "server" {}

resource "local_file" "inventory" {
  content  = "Server name: ${random_pet.server.id}"
  filename = "/tmp/server.txt"
}
```
Here, `random_pet.server.id` is an exported attribute of the `random_pet` resource. Terraform automatically infers that `local_file.inventory` depends on `random_pet.server` and will create the pet first. That implicit graph edge is what makes Terraform feel declarative instead of procedural.

### 4b.2: Types of Named Values

The documentation lists several named values you can reference:

Prefix|Meaning|Example
-|-|-
`var.`|Input variable|`var.instance_count`
`local.`|Local value|`local.common_tags`
`module.`|Child module outputs|`module.vpc.vpc_id`
`data.`|Data source attributes|`data.aws_ami.ubuntu.id`
`path.module`|Filesystem path of the module|`${path.module}/files`
`terraform.workspace`|Current workspace name|`terraform.workspace`
`count.index`	|Index in `count` loop	|`count.index`
`each.key` / `each.value`	|Key/value in `for_each`	|`each.value`


### 4b.3: Resource Addresses

A resource address uniquely identifies a resource instance. The documentation's Resource Address Reference section details the format:

```text
[module.MODULE_NAME[INDEX].]RESOURCE_TYPE.RESOURCE_NAME[INSTANCE_INDEX]
```

**Examples:**

- `aws_instance.web` — Single instance.

- `aws_instance.web[0]` — First instance if `count = 3`.

- `aws_instance.web["key"]` — Instance for key `"key"` if `for_each` used.

- `module.vpc.aws_subnet.private` — Resource in a child module.

**Important Note:** Resource addresses are used with commands like `terraform state list`, `terraform state rm`, `terraform taint` (deprecated), and `terraform plan -replace=ADDRESS`. Think of the address as Terraform's exact pointer to one object in the graph and in state.

### 4b.4: Splat Expressions

When a resource uses count or `for_each`, it represents multiple instances. To get a list of an attribute across all instances, use the splat operator `[*]`.

```hcl
resource "aws_instance" "web" {
  count = 3
  ami   = "ami-123"
}

output "all_ips" {
  value = aws_instance.web[*].public_ip
}
```

This will output a list of public IPs for all three instances.
For `for_each` which produces a map, splat does not work directly. Use `values()`:

```hcl
output "all_ips" {
  value = values(aws_instance.web)[*].public_ip
}
```

### 4b.5: Implicit vs. Explicit Dependencies

**Implicit Dependency:** Terraform detects references and automatically builds the graph.

```hcl
resource "aws_security_group" "allow_ssh" {
  # ...
}

resource "aws_instance" "web" {
  vpc_security_group_ids = [aws_security_group.allow_ssh.id]
  # Terraform knows instance depends on SG
}
```

**Explicit Dependency:** Use `depends_on` when the relationship isn't captured by attribute references (e.g., application-level dependencies, or when a resource needs to wait for another resource's side effect that isn't an exported attribute).

```hcl
resource "aws_s3_bucket" "data" { ... }

resource "aws_instance" "app" {
  # No direct attribute reference to the bucket
  depends_on = [aws_s3_bucket.data]
}
```

**Best Practice:** Rely on implicit dependencies whenever possible. Use explicit depends_on only when necessary to avoid unnecessary coupling or when the dependency is not captured by references.

### 4b.6: Hands-On Lab — References and Dependencies

Directory: `obj4b-lab`

File: `main.tf`

```hcl
terraform {
  required_providers {
    random = { source = "hashicorp/random", version = "~> 3.6" }
    local  = { source = "hashicorp/local", version = "~> 2.5" }
  }
}

resource "random_pet" "prefix" {
  length = 2
}

resource "local_file" "config" {
  filename = "${path.module}/${random_pet.prefix.id}.cfg"
  content  = "setting=value"
  # Implicit dependency on random_pet.prefix
}

resource "local_file" "readme" {
  filename = "${path.module}/README.txt"
  content  = "Configuration file: ${local_file.config.filename}"
  # Implicit dependency on local_file.config
}

# Explicit dependency for no apparent reason (demonstration)
resource "local_file" "metadata" {
  filename = "${path.module}/metadata.json"
  content  = jsonencode({ created_by = "Terraform" })
  depends_on = [local_file.readme] # Forces metadata to be created after readme
}

output "config_file" {
  value = local_file.config.filename
}
```

#### Steps:

1. `terraform init`

2. `terraform graph | dot -Tpng > graph.png (requires Graphviz) to visualize dependencies.

3. `terraform apply` — Observe creation order.

Examine `terraform.tfstate` to see the dependencies arrays.

### 4b.7: Mini-Quiz - References and Dependencies

1. True/False: A resource's local name can be used to reference that resource from a different module.

2. Multiple Choice: Given `resource "aws_instance" "web" { count = 2 }`, which expression returns a list of both instance IDs?
```
A. aws_instance.web.id
B. aws_instance.web[*].id
C. aws_instance.web[0].id
D. aws_instance.web.*.id
```

3. Multiple Answer: Which of the following create a dependency? **(Select TWO)**
```
A. ami = var.ami_id
B. subnet_id = aws_subnet.main.id
C. depends_on = [aws_vpc.main]
D. tags = { Name = "web" }
```

<details>

<summary>Show answers</summary>

**Answers & brief explanations**

1. **FALSE** — A resource's local name (e.g., `"web"`) is scoped to the module where it is declared. To reference a resource from a different module, you must use that module's output value, not the resource’s local name directly.

2. **B** — `aws_instance.web[*].id` returns a list of both instance IDs. This is the modern splat expression syntax. Option D (aws_instance.web.*.id) is the older legacy syntax, still valid but less preferred; however, B is the idiomatic answer. (The question asks for the expression that returns a list; both B and D technically work, but the recommended and unambiguous answer is B.)

3. **B, C** — B creates an implicit dependency because it references an attribute of another resource. C creates an explicit dependency using depends_on. A and D do not create dependencies on other resources.

</details>


## Objective 4c: Use variables and outputs

*Source: [`Customize Terraform configuration with variables`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/variables), [`Use input variables to add module arguments`](https://developer.hashicorp.com/terraform/language/v1.12.x/values), [`Output data from Terraform`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/outputs), [`Output Block Reference`](https://developer.hashicorp.com/terraform/language/v1.12.x/block/output)*

### 4c.1: Input Variables (`variable` Block)

Variables parameterize your configuration. They are defined using the `variable` block:

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t2.micro"

  # Optional validation
  validation {
    condition     = can(regex("^t[23]\\..+", var.instance_type))
    error_message = "Instance type must be t2 or t3 series."
  }

  # Mark as sensitive (redacts from CLI output)
  sensitive = true
}
```

**Arguments:**

- `type`: Constrains the expected type (``string`, `number`, `bool`, `list(...)`, `map(...)`, `object(...}`, etc.).

- `description`: Documentation for the variable.

- `default`: If provided, the variable becomes optional. If omitted, the variable is required.

- `validation`: Custom condition to validate input.

- `sensitive`: Hides value from CLI output (but still in state!).

- `nullable`: (Terraform 1.1+) Controls whether `null` is allowed.

**Variable Precedence:**
When a value is assigned multiple ways, Terraform uses this order (highest to lowest):

1. Command-line `-var` or `-var-file`

2. `*.auto.tfvars` or `*.auto.tfvars.json`

3. `terraform.tfvars.json`

4. `terraform.tfvars`

5. Environment variables (`TF_VAR_name`)

6. `default` in the `variable` block

If none provided and no default, Terraform prompts interactively.

### 4c.2: Local Values (`locals` Block)

Locals are internal constants or computed values within a module. They are not exposed externally.

```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
  name_prefix = "${var.project}-${terraform.workspace}"
}

resource "aws_instance" "web" {
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-web" })
}
```

**Best Practice:** Use locals to avoid repeating expressions and to make the configuration more readable.

### 4c.3: Output Values (`output` Block)

Outputs expose information about your infrastructure. They are printed after `apply` and can be queried with `terraform output`.

```hcl
output "instance_public_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of the web server"
  sensitive   = true  # Redacts from CLI unless specifically requested
}
```

**Key Points:**

- Root module outputs are displayed in the CLI and stored in state.

- Child module outputs can be accessed by the parent module via `module.MODULE_NAME.OUTPUT_NAME`.

- Marking an output `sensitive` redacts it from the CLI unless you query it specifically by name (`terraform output instance_public_ip`).

- Sensitive outputs are still stored in plaintext in the state file.

### 4c.4: Variable Definition Files

Files with `.tfvars` or `.auto.tfvars` extension are automatically loaded. They use simple HCL syntax:

```hcl
# terraform.tfvars
instance_type = "t3.medium"
environment   = "prod"

tags = {
  Owner = "Platform Team"
  CostCenter = "12345"
}
```

**Important Note:** Never commit `.tfvars` files containing secrets. Use remote variable storage (HCP Terraform, Vault) or environment variables.

### 4c.5: Hands-On Lab — Variables and Outputs

Directory: `obj4c-lab`

File:  `variables.tf`

```hcl
variable "file_name" {
  type        = string
  description = "Name of the file to create"
  default     = "default.txt"
}

variable "file_content" {
  type        = string
  description = "Content to write"
  sensitive   = true
}

variable "append_timestamp" {
  type    = bool
  default = false
}
```

File: `terraform.tfvars` (create this; it will be ignored by .gitignore)

```hcl
file_name = "custom.txt"
append_timestamp = true
```

File: `main.tf`

```hcl
locals {
  final_content = var.append_timestamp ? "${var.file_content}\nGenerated at: ${timestamp()}" : var.file_content
}

resource "local_file" "output" {
  filename = "${path.module}/${var.file_name}"
  content  = local.final_content
}

output "file_path" {
  value = local_file.output.filename
}

output "file_content" {
  value     = local_file.output.content
  sensitive = true
}
```

#### Steps:

1. `terraform init`

2. `terraform plan -var="file_content=Secret123"` — Note that file_content is redacted in plan.

3. `terraform apply -var="file_content=Secret123" -auto-approve`

4. `terraform output` — file_content is redacted.

5. `terraform output -json` — file_content is exposed! This demonstrates that sensitive is only a UI redaction.

6. `terraform output file_content` — Shows the actual content.

### 4c.6: Mini-Quiz - Variables and Outputs

1. **True/False:** If a variable has no default and no value is provided, Terraform will use null and proceed with the plan.

2. **Multiple Choice:** Which variable value source has the highest precedence?

```
A. Environment variable TF_VAR_name
B. terraform.tfvars
C. Command-line -var 'name=value'
D. Default value in variable block
```

3. **Multiple Answer:** Which statements about output values are true? **(Select TWO)**
```
A. Outputs from child modules are automatically displayed in the root module's output.
B. Outputs marked sensitive are encrypted in the state file.
C. Outputs can be queried individually with `terraform output NAME`.
D. Outputs are stored in the state file and can be accessed by other configurations via `terraform_remote_state`.
```

<details>

<summary>Show answers</summary>

**Answers & brief explanations**

1. **False.** If a variable has no default and no value is provided, Terraform does not silently continue with `null` in the general case. It prompts for a value in interactive mode or fails in non-interactive mode unless the variable is explicitly allowed to be null and the expression can handle it.

2. **C.** The command-line `-var` flag has the highest precedence. It wins over environment variables, `tfvars` files, and defaults because it is the most explicit value source.

3. **C, D.** Outputs can be queried individually with `terraform output NAME`, and outputs are stored in state so other configurations can read them via `terraform_remote_state`. Child module outputs are not automatically shown in the root unless the root re-exports them, and sensitive outputs are redacted in the CLI but not encrypted in state.


</details>


## Objective 4d: Understand and use complex types

*Source: [`Complex types`](https://developer.hashicorp.com/terraform/language/v1.12.x/expressions/type-constraints#complex-types), [`Customize Terraform configuration with variables`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/variables)*

### 4d.1: Primitive Types

- string: Text, e.g., "hello"

- number: Integer or float, e.g., 42 or 3.14

- bool: true or false

Terraform will automatically convert between these when possible (e.g., "42" to 42).

### 4d.2: Collection Types

- list(<TYPE>): Ordered sequence, e.g., list(string) for ["a", "b", "c"].

- map(<TYPE>): Key-value pairs where keys are strings, e.g., map(string) for { key1 = "val1", key2 = "val2" }.

- set(<TYPE>): Unordered collection of unique values, e.g., set(string) for ["a", "b"] (duplicates removed).

**Important Note:** list and map without type argument are legacy shorthands for list(any) and map(any). The exam prefers explicit types.

### 4d.3: Structural Types

object({ ATTR = TYPE, ... }): Defines a structure with specific attributes of specific types.

```hcl
variable "server" {
  type = object({
    name = string
    cpu  = number
    tags = map(string)
  })
}
tuple([TYPE, TYPE, ...]): Fixed-length list with specific types per index.

```hcl
variable "endpoint" {
  type = tuple([string, number])  # e.g., ["example.com", 443]
}
```

### 4d.4: Type Conversion and any

Terraform automatically converts between compatible types where possible.

any is a placeholder that Terraform resolves to a concrete type based on the provided value. Use sparingly—only when you truly don't care about the type (e.g., passing data directly to jsonencode).

### 4d.5: Optional Object Attributes

Terraform 1.3+ supports optional attributes in object types:

```hcl
variable "config" {
  type = object({
    name    = string
    enabled = optional(bool, true)
    tags    = optional(map(string), {})
  })
}
```
If the caller omits enabled, it defaults to true.

### 4d.6: Hands-On Lab — Complex Types

Directory: obj4d-lab

File: `variables.tf`

```hcl
variable "servers" {
  type = list(object({
    name = string
    size = string
    tags = optional(map(string), {})
  }))
  default = [
    { name = "web", size = "small" },
    { name = "db", size = "large", tags = { backup = "true" } }
  ]
}

variable "regions" {
  type = set(string)
  default = ["us-east-1", "us-west-2"]
}
```

File: `main.tf`

```hcl
locals {
  server_list = join(", ", [for s in var.servers : "${s.name} (${s.size})"])
}

resource "local_file" "summary" {
  filename = "${path.module}/servers.txt"
  content  = <<-EOT
  Servers: ${local.server_list}
  Regions: ${join(", ", var.regions)}
  EOT
}

output "first_server_name" {
  value = var.servers[0].name
}
```

Steps:

1. `terraform init`

2. `terraform apply -auto-approve`

3. `terraform show` to inspect `servers.txt` to see the rendered list.

Try overriding with a terraform.tfvars that provides a different servers list.

### 4d.7: Mini-Quiz - Complex Types
True/False: A tuple type requires all elements to be of the same type.

Multiple Choice: Which type would you use for a variable that expects a map where all values are strings?
A. map
B. map(string)
C. object(string)
D. list(string)

Multiple Answer: Given variable "data" { type = map(any) }, which assignments are valid? (Select THREE)
A. { a = "one", b = 2 }
B. { a = "one", b = "two" }
C. ["one", "two"]
D. { nested = { x = 1 } }

<details>

<summary>Show answers</summary>

**Answers & brief explanations**

1. **FALSE** — A tuple allows elements of different types and has a fixed length. A list is homogeneous, while a tuple is positional and can mix types by index.

2. **Multiple Choice: B** — `map(string)` declares a map where all values are strings. `map` alone is a shorthand for `map(any)`, which is less strict. `object(string)` is invalid because object types need named attributes, and `list(string)` is a list, not a map.

3. **Multiple Answer: A, B, D** — `map(any)` accepts maps where values can be different types. `{ a = "one", b = 2 }`, `{ a = "one", b = "two" }`, and `{ nested = { x = 1 } }` are all valid. `map(any)` is intentionally flexible, so nested maps are allowed too.

</details>

> That was a lot of content! Take a moment to review and ensure you understand the concepts before moving on to the quiz.

## Objective 4 Part 1: Quiz


**Question 1 (True/False):**

A data block can be configured with count to fetch multiple instances of a data source.

**Question 2 (Multiple Choice):**

You need to reference the private IP address of an AWS instance defined as `resource "aws_instance" "app"`. Which expression is correct?
```
⬜ aws_instance.app.private_ip
⬜ resource.aws_instance.app.private_ip
⬜ data.aws_instance.app.private_ip
⬜ var.aws_instance.app.private_ip
````

**Question 3 (Multiple Answer):**

Which of the following are valid meta-arguments for a resource block? **(Select THREE)**
```
⬜ depends_on
⬜ sensitive
⬜ lifecycle
⬜ provider
⬜ validation
```

**Question 4 (True/False):**

A local value `(locals block)` can reference other local values defined later in the same block, as long as there is no circular dependency.

**Question 5 (Multiple Choice):**
You have a variable defined as:

```hcl
variable "instance_count" {
  type    = number
  default = 1
}
```

You set an environment variable `TF_VAR_instance_count=3` and also pass `-var="instance_count=5"` on the command line. What value will Terraform use?
```
⬜ 1
⬜ 3
⬜ 5
⬜ Terraform will error due to conflicting values.
```

**Question 6 (Multiple Choice):**

Which type constraint would correctly validate the value `{ id = "i-123", tags = { Name = "web" } }`?
```
⬜ map(string)
⬜ object({ id = string, tags = map(string) })
⬜ list(string)
⬜ tuple([string, map(string)])
```
**Question 7 (True/False):**

Output values marked as sensitive are automatically encrypted in the state file.

**Question 8 (Multiple Choice):**

You have resource `aws_instance "web" { count = 3 }`. You want to output a list of all instance IDs. Which output expression is correct?
```
⬜ value = aws_instance.web.id
⬜ value = aws_instance.web[*].id
⬜ value = aws_instance.web[0].id
⬜ value = aws_instance.web.*.id
```
**Question 9 (Multiple Answer):**

Which of the following statements about data sources are true? (Select THREE)
```
⬜ Data sources can be used to read existing infrastructure without managing it.
⬜ Data sources are read during terraform plan if all arguments are known.
⬜ Data sources can be modified by terraform apply.
⬜ Data sources support the depends_on meta-argument.
```
**Question 10 (True/False):**

A child module's output can be accessed by the root module using the syntax `module.MODULE_NAME.OUTPUT_NAME`.

**Question 11 (Multiple Choice):**

You want to ensure that a `variable` environment only accepts the values `"dev"`, `"staging"`, or `"prod"`. What is the best way to enforce this?
```
⬜ Set type = string and rely on the provider to fail if invalid.
⬜ Add a validation block with a contains check.
⬜ Set sensitive = true to hide the value.
⬜ Use a locals block to map the value.
```
**Question 12 (Multiple Choice):**

What is the correct way to reference a data source of type `aws_ami` with local name `ubuntu`?
```
⬜ resource.aws_ami.ubuntu.id
⬜ data.aws_ami.ubuntu.id
⬜ aws_ami.ubuntu.data.id
⬜ var.aws_ami.ubuntu.id
```
**Question 13 (True/False):**

A set(string) variable type ensures that elements are unique and order is preserved.

**Question 14 (Multiple Choice):**

You define variable `"ports" { type = list(number) }`. Which value assignment is invalid?
```
⬜ [80, 443]
⬜ [80, "443"]
⬜ [8080]
⬜ []
```
**Question 15 (Multiple Answer):**

Which of the following are valid ways to assign a value to an input variable? **(Select FOUR)**
```
⬜ Defining a default in the variable block.
⬜ Creating a terraform.tfvars file.
⬜ Hardcoding the value in the resource block.
⬜ Using the -var command-line flag.
⬜ Setting an environment variable named TF_VAR_name
```

<details>
<summary>Show answers</summary>

**Answers & brief explanations**

**Q1: TRUE** — The `count` meta‑argument is supported on `data` blocks to fetch multiple instances of a data source.

**Q2: A** — `aws_instance.app.private_ip` is the correct reference. The format is `<resource_type>.<local_name>.<attribute>`.

**Q3: A, C, D** — `depends_on`, `lifecycle`, and `provider` are valid resource meta‑arguments. sensitive is for variables/outputs, not resources. `validation` is for variable blocks.

**Q4: TRUE** — Local values can reference other local values defined later in the same locals block as long as there is no circular dependency. Terraform resolves the local expressions as a graph, so the order in the block is not the controlling factor.

**Q5: C** — Command‑line `-var` (5) has the highest precedence, overriding the environment variable (3) and the default (1).

**Q6: B** — `object({ id = string, tags = map(string) })` matches the structure exactly. `map(string)` would not enforce the nested structure; `list(string)` and `tuple` do not match.

**Q7: FALSE** — Sensitive outputs are not encrypted in the state file. The state file stores them in plain text, but Terraform redacts them in CLI output.

**Q8: B** — `aws_instance.web[*].id` is the correct splat expression to get a list of all instance IDs. `aws_instance.web.*.id` (D) is an older legacy syntax; `[*]` is the modern and recommended way.

**Q9: A, B, D** — Data sources can read existing infrastructure without managing it, they are read during `plan` when all arguments are known, and they support `depends_on` when you need to force the read to wait. They do not modify infrastructure; `terraform apply` only evaluates the read result.

**Q10: TRUE** — The syntax `module.MODULE_NAME.OUTPUT_NAME` is exactly how the root module reads a child module output.

**Q11: B** — A `validation` block with `contains()` or a regular expression is the best way to enforce allowed values. Relying on the provider (A) could cause errors late; sensitive (C) hides but doesn't validate; locals (D) does not enforce at input.

**Q12: B** — `data.aws_ami.ubuntu.id` is the correct reference. Data sources always use the `data` prefix, followed by the data type, local name, and attribute.

**Q13: FALSE** — A `set(string)` ensures uniqueness but does not preserve order. Order is not guaranteed for sets.

**Q14: B** — `[80, "443"]` is invalid because the type is `list(number)`, so all elements must be numbers. The string "443" is not a number.

**Q15: A, B, D, E** — Defining a default (A), creating a `terraform.tfvars` file (B), using the `-var` flag (D), and setting `TF_VAR_name` in the environment (E) are all valid ways to assign input variable values. Hardcoding in a resource block bypasses the variable entirely.

</details>


## Objective 4 Part 2

That wraps the first half of Objective 4. If configuration were a skyscraper, part 1 is the foundation, the framing, and the wiring. Part 2 is where we add the elevators, glass, and all the things that make it look impressive from the outside.

Light monumental joke: Terraform part 2 is coming in like a stone monument with a README - solid, large, and impossible to ignore.

Lets move on to [Objective 4 (Part-2): Write Terraform configuration using HCL](./04-Terraform-configuration-part-2.md) where we will explore the syntax and structure of Terraform's configuration language, HCL.