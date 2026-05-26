# Objective 6: Terraform State Management

> This objective moves beyond writing code and into the operational reality of Terraform. State is the source of truth, the memory of your infrastructure. Mismanage it, and you will experience production outages that no amount of clever `HCL` can fix.

Below is the outline of **Objective 6**. We will cover each part in depth, with explanations, examples, and hands-on labs. You can jump to any section directly, but I recommend following the order for the best learning experience.

### 6.0: The Terraform State Mental Model

Keep these three things separate:

- **Configuration** describes the desired end state.
- **State** is Terraform's memory of what it believes already exists.
- **Real infrastructure** is the actual cloud resource or local object.

*Terraform compares configuration to state, then refreshes state from reality. If state drifts away from reality, Terraform can make the wrong choices even when the configuration is correct. That is why state must be treated as operational data, not disposable scratch space.*

The practical rule: prefer declarative refactors (`moved`, `import`, `refresh-only`) over manual state editing.

- [**Objective 6a:** Describe the local backend](#objective-6a-describe-the-local-backend)

- [**Objective 6b:** Describe state locking](#objective-6b-describe-state-locking)

- [**Objective 6c:** Configure remote state using the backend block](#objective-6c-configure-remote-state-using-the-backend-block)

- [**Objective 6d:** Manage resource drift and Terraform state](#objective-6d-manage-resource-drift-and-terraform-state)

- [**Objective 6: Quiz**](#objective-6-quiz)

---

## Objective 6a: Describe the local backend

*Source: [`Initialize Terraform Configuration`](https://developer.hashicorp.com/terraform/tutorials/cli/init), [`Backend Type local`](https://developer.hashicorp.com/terraform/language/v1.12.x/backend/local), [`Backend block configuration overview`](https://developer.hashicorp.com/terraform/language/v1.12.x/backend)*

### 6a.1: What is a Backend?

A backend in Terraform determines two things:

1. **Where state is stored.**
2. **How Terraform performs operations** (locally or remotely).

Every Terraform configuration has exactly one backend. If you do not explicitly configure one, Terraform uses the **local backend**.

### 6a.2: The Local Backend: Default Behavior

The local backend stores state in a plain JSON file named `terraform.tfstate` in the root directory of your configuration. This is the simplest setup and is perfectly fine for a **single developer** working on a personal project.

**What Terraform does with the local backend:**

1. During `terraform plan`, it reads `terraform.tfstate` from the current directory.
2. During `terraform apply`, it updates `terraform.tfstate` with the new state.
3. Before updating, it copies the existing state to `terraform.tfstate.backup` as a safety measure.

**Key files created by the local backend:**

- `terraform.tfstate` — The primary state file. Contains all resource metadata in JSON.
- `terraform.tfstate.backup` — The previous state, saved before the last `apply`. Used for recovery if state is corrupted during an apply.

**Example local backend configuration (explicit, though default):**
```hcl
# This is what Terraform implicitly uses when no backend is specified.
# You rarely need to write this explicitly. We write it here to understand it.

terraform {
  # The backend block defines where state is stored
  backend "local" {
    # 'path' is the only argument. Defaults to "terraform.tfstate"
    path = "terraform.tfstate"
  }
}
```

**Explanation of the local backend configuration:**

- `terraform {` — The top-level block for configuring Terraform itself (not infrastructure).
- `backend "local" {` — Declares that we want to use the `local` backend type. `"local"` is the backend type identifier.
- `path = "terraform.tfstate"` — Specifies the exact file path where the state JSON will be written. This is relative to the root module directory. If omitted, it defaults to `terraform.tfstate`.

### 6a.3: Limitations of the Local Backend

The local backend has critical limitations for team use:

| Limitation | Consequence |
|------------|-------------|
| **No state locking** | Terraform has no shared lock service in the local backend, so separate copies of the same state can be modified concurrently and overwrite each other. |
| **State stored on disk** | The state file sits on a developer's laptop. Disk failures, accidental deletion, or `git clean` can destroy the only record of your infrastructure. |
| **No encryption at rest** | The state file contains all resource attributes, including secrets, in plaintext on disk. Anyone with filesystem access can read them. |
| **No collaboration** | Team members cannot share state. Each person has their own copy, leading to divergence. |

**The local backend is suitable only for:**

- Learning and experimentation.
- Single-developer personal projects.
- Scenarios where infrastructure is disposable and no collaboration is needed.

### 6a.4: What's Inside the State File?

Let's examine the structure of `terraform.tfstate` to understand what we're protecting.

```json
{
  "version": 4,
  "terraform_version": "1.12.0",
  "serial": 3,
  "lineage": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
  "outputs": {
    "instance_id": {
      "value": "i-1234567890abcdef0",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "ami": "ami-0c55b159cbfafe1f0",
            "id": "i-1234567890abcdef0",
            "instance_type": "t3.micro",
            "private_ip": "10.0.1.42",
            "tags": {
              "Name": "web-server"
            },
            "password": "MySecretPassword123"
          },
          "dependencies": [
            "aws_subnet.main",
            "aws_security_group.allow_ssh"
          ]
        }
      ]
    }
  ]
}
```

**Key fields explained:**

- `version` — The state file format version. Terraform increments this when the state schema changes.
- `terraform_version` — The version of Terraform that last wrote this state. If you try to read this state with an older Terraform that doesn't support this format, it will error.
- `serial` — A monotonically increasing integer that records the version of the state. Each successful state write increments it. Remote backends use it as part of concurrency safety and optimistic checks, but it is not the same thing as a lock.
- `lineage` — A UUID generated when the state is first created. This uniquely identifies the "ancestry" of the state. If you lose state and try to recreate it from scratch, the new state will have a different lineage, and Terraform will refuse to mix them unless you explicitly override.
- `outputs` — Contains all root module output values, stored in plaintext.
- `resources` — An array of all managed resources. Each entry includes the resource type, local name, provider, and one or more instances (for `count`/`for_each`).
- `instances[].attributes` — All the arguments and exported attributes of the resource, including sensitive ones. **Note the `password` field—it's in plaintext.**
- `instances[].dependencies` — The list of resource addresses that this resource depends on. This is used to determine the correct order for destruction.

### 6a.5: Hands-On Lab — Inspecting Local State

**Directory:** `obj6a-lab`

**File: `main.tf`** — We'll create a resource that generates a "secret" so we can find it in state.
```hcl
# Configures the required provider for this configuration.
# Without this block, terraform init won't know what to download.
terraform {
  required_providers {
    random = {
      source  = "hashicorp/random"   # The registry address of the provider
      version = "~> 3.6.0"            # Accept any 3.6.x version, but not 3.7
    }
  }
}

# This resource generates a random string. We give it a local name "password"
# so we can reference it elsewhere (e.g., random_password.password.result).
# The actual generated value will be stored in the state file in plaintext.
resource "random_password" "password" {
  length  = 16          # Generate 16 characters
  special = false       # Don't include special characters like !@#$
}

# This output prints the generated password to the CLI after apply.
# It is marked sensitive so Terraform redacts it in the terminal output,
# but the value is STILL stored in plaintext in terraform.tfstate.
output "generated_password" {
  value     = random_password.password.result
  sensitive = true        # Redacts from CLI output only, NOT from state file
}
```

**Steps:**

1. `terraform init` — Downloads the `random` provider into `.terraform/providers/`.
2. `terraform apply -auto-approve` — Creates the random password. The output is marked sensitive, so you'll see `(sensitive value)` in the terminal.
3. Open `terraform.tfstate` in VS Code. Search for `"result"`. You will see the **actual generated password in plaintext** inside the JSON.
4. Note the `serial` value (probably 1). Note the `lineage` value.
5. Run `terraform plan` or `terraform apply -auto-approve` again. If there are no changes, the state serial should remain unchanged because Terraform does not rewrite state when nothing changed.
6. Note that `terraform.tfstate.backup` also exists. This is the state from BEFORE the last apply. If your state gets corrupted, you can copy this file over `terraform.tfstate` to revert to the previous known-good state.

**Key takeaway:** The `sensitive = true` flag on a variable or output **redacts the value from CLI output only**. It does **not** encrypt the value in the state file. The state file is a plaintext JSON document.

---

### 6a.6: Mini-Quiz for 6a

1. True/False: The local backend supports automatic state locking to prevent concurrent modifications.
2. Multiple Choice: What is the purpose of the `lineage` field in a state file?
   A. It tracks the number of times `apply` has been run.
   B. It uniquely identifies the state's history to prevent mixing state from different configurations.
   C. It stores the Terraform version that generated the state.
   D. It records the user who last ran `terraform apply`.
3. True/False: Marking a variable as `sensitive = true` encrypts its value in the `terraform.tfstate` file.

<details>

<summary><strong>Show Answers</strong></summary>

**Answers & brief explanations**

1. **False.** The local backend does not support state locking. This means that if two people run `terraform apply` at the same time from different machines, they can both read the same state and make changes, but the second one to write will overwrite the first, causing lost changes and corrupted state.
2. **B.** The `lineage` field is a UUID that uniquely identifies the history of the state. If you lose your state and try to recreate it from scratch, the new state will have a different lineage. Terraform uses this to prevent mixing state from different configurations, which could lead to resource mismanagement.
3. **False.** The `sensitive = true` flag on a variable or output only redacts the value from CLI output. It does not encrypt the value in the state file. The state file is a plaintext JSON document, and all values, including sensitive ones, are stored in plaintext.

</details>

## Objective 6b: Describe state locking

*Source: [`State Locking`](https://developer.hashicorp.com/terraform/language/v1.12.x/state/locking) [`State Storage and Locking`](https://developer.hashicorp.com/terraform/language/v1.12.x/state/backends)*

### 6b.1: The Concurrency Problem

Imagine two DevOps engineers, Alice and Bob, both responsible for the same AWS infrastructure. They both have the Terraform configuration on their laptops.

1. Alice runs `terraform plan`. Terraform reads the state from an S3 bucket. The state says there's one EC2 instance.
2. Bob also runs `terraform plan` at the exact same time. Terraform reads the same state from the S3 bucket. The state also shows one EC2 instance.
3. Alice runs `terraform apply`. She adds a new EC2 instance. Terraform writes the new state (with two instances) to S3. **Serial changes from 10 to 11.**
4. Bob runs `terraform apply`. His local state still says one instance. Terraform creates another new instance. It writes the state to S3, **overwriting Alice's state with a version that only has Bob's new instance. Serial 11.**

Now:

- The S3 state file shows Bob's version (two instances: original + Bob's).
- In AWS, there are **three** instances (original + Alice's + Bob's).
- Alice's instance still exists, but Terraform no longer knows about it. It is **orphaned**. Terraform won't manage it, update it, or destroy it. This is a disaster.

**State locking prevents this scenario entirely.**

---

### 6b.2: What is State Locking?

State locking is a mechanism that ensures **only one person** (or CI pipeline) can modify the state at any given time.

Common control flags:

- `-lock-timeout=10m` tells Terraform how long to keep retrying before giving up on a locked state.
- `-lock=false` disables locking entirely. This is sometimes used for emergency reads or troubleshooting, but it is dangerous for normal apply/destroy workflows because it can reintroduce the very race conditions locking is meant to prevent.

**How it works:**

1. Before any operation that could modify state (`plan` with default refresh, `apply`, `destroy`), Terraform **acquires a lock** on the state.
2. If the lock is already held by another process, Terraform waits (for a configurable timeout) or immediately errors, depending on configuration.
3. Once the lock is acquired, Terraform proceeds with the operation.
4. After the operation completes (success or failure), Terraform **releases the lock**.

**If two people try to apply simultaneously:**

1. Alice runs `terraform apply`. Terraform acquires the lock. Lock holder: Alice.
2. Bob runs `terraform apply`. Terraform tries to acquire the lock. It sees the lock is held by Alice. Bob's Terraform immediately errors: `Error acquiring the state lock`. Bob must wait for Alice to finish.
3. Alice's apply completes. Terraform releases the lock.
4. Bob can now run `terraform apply` again. This time, Terraform acquires the lock and proceeds. Bob's apply reads the state that includes Alice's changes, so his plan is correct.

---

### 6b.3: How Locks Work in Different Backends

Not all backends support locking. The local backend has no locking at all (no coordination between different laptops).

| Backend Type | Locking Support | Mechanism |
|--------------|:---:|-----------|
| **local** | ❌ No | No shared lock coordination |
| **s3** | ✅ Yes (when configured) | DynamoDB table. Terraform writes a lock record to DynamoDB before proceeding. The record includes the operation type, timestamp, and user. |
| **gcs** | ✅ Yes | Object generation and preconditions. GCS uses object metadata to track locks. |
| **azurerm** | ✅ Yes | Azure Blob Storage leases. Terraform acquires a lease on the blob containing the state file. |
| **consul** | ✅ Yes | Consul's built-in session and lock mechanism. |
| **HCP Terraform** | ✅ Yes | Built into the platform. No additional configuration needed. |

---

### 6b.4: Configuring Locking with S3 + DynamoDB

This is the most common remote backend for teams not using HCP Terraform.

```hcl
# This backend block tells Terraform to store state in an S3 bucket
# and use a DynamoDB table for state locking.
terraform {
  backend "s3" {
    # The S3 bucket where state will be stored
    bucket = "my-company-terraform-state"

    # The path/key within the bucket. Use a naming convention that reflects
    # the project and environment (e.g., "prod/network/terraform.tfstate")
    key = "prod/app/terraform.tfstate"

    # The AWS region where both the bucket and DynamoDB table exist
    region = "us-east-1"

    # The DynamoDB table used for state locking. Terraform will create/use
    # an item in this table with the same key as the state file.
    dynamodb_table = "terraform-locks"

    # Encrypt the state file at rest using S3 server-side encryption (AES-256)
    encrypt = true
  }
}
```

**Explanation:**

- `backend "s3" {` — Specifies the backend type as `s3`.
- `bucket = "my-company-terraform-state"` — The name of the S3 bucket. This bucket must already exist.
- `key = "prod/app/terraform.tfstate"` — The S3 object key (path). This uniquely identifies the state file for this specific configuration within the bucket.
- `region = "us-east-1"` — The AWS region where both the bucket and DynamoDB table are located.
- `dynamodb_table = "terraform-locks"` — The name of the DynamoDB table. This table must already exist with a primary key named `LockID` (of type String). Terraform uses this table to create lock records.
- `encrypt = true` — Enables AES-256 server-side encryption on the S3 object. Protects the state file from being read directly from S3 by unauthorized parties.

**How the DynamoDB lock works step-by-step:**

1. Terraform writes a new item to the DynamoDB table with `LockID = "prod/app/terraform.tfstate"` and a condition that the item doesn't already exist (or the existing lock has expired).
2. If the item already exists (someone else holds the lock), DynamoDB returns a `ConditionalCheckFailedException`. Terraform interprets this as "lock already held" and waits or errors.
3. When the operation completes, Terraform deletes the item from DynamoDB, releasing the lock.

---

### 6b.5: Hands-On Lab — Observing Lock Behavior (Simulation)

We can't easily set up S3 and DynamoDB locally, but we can demonstrate the concept by trying to run two `apply` operations on the same local state. The local backend uses file advisory locks, which might block on the same machine.

**Alternative approach:** We'll create two identical configurations in separate directories that share a "remote" state via a local file that simulates a remote backend. This isn't a true remote backend but demonstrates collision.

Since this is a bit contrived, I'll focus on a clear conceptual demonstration instead.

**Conceptual lab — Simulating a race condition:**

1. Create directory `obj6b-lab-a` with a Terraform config that creates a `local_file`.
2. Run `terraform apply` and note the state serial.
3. Manually copy the state file to `obj6b-lab-b`.
4. In `obj6b-lab-a`, modify the config to create a second file. Run `terraform apply`. Serial increments.
5. In `obj6b-lab-b`, also modify the config to create a different second file. Run `terraform apply`.
6. Observe that `obj6b-lab-b`'s apply succeeds because it's using its own local state, not the updated state from `obj6b-lab-a`. The two directories have diverged.

**Key takeaway:** Without a shared remote backend with locking, state divergence is inevitable.

---

### 6b.6: Mini-Quiz for 6b

1. True/False: State locking is automatically enabled when using the `s3` backend without any additional configuration.
2. Multiple Choice: In the S3 + DynamoDB setup, what is the primary key of the DynamoDB table used for locking?
   A. `StateKey`
   B. `LockID`
   C. `SerialNumber`
   D. `LineageUUID`
3. Multiple Answer: Which of the following backends support state locking? (Select THREE)
   A. `local`
   B. `s3` with DynamoDB
   C. `gcs`
   D. `http`
   E. `azurerm`

<details>
<summary><strong>Show Answers</strong></summary>

1. **False.** The `s3` backend does not lock by itself. You only get locking when you also configure DynamoDB for the lock record.

2. **B.** `LockID` is the required primary key for the DynamoDB lock table. Terraform uses that key to create and release locks for the state path.

3. **B, C, E.** `s3` with DynamoDB, `gcs`, and `azurerm` support shared state locking. The local backend and `http` backend do not provide that same lock coordination.

</details>

## Objective 6c: Configure remote state using the backend block

*Source: [`Backend block configuration overview`](https://developer.hashicorp.com/terraform/language/v1.12.x/backend), [`State Storage and Locking`](https://developer.hashicorp.com/terraform/language/v1.12.x/state/backends)*

### 6c.1: The `backend` Block Syntax

We've seen the S3 backend. Let's look at the general structure.

```hcl
terraform {
  backend "BACKEND_TYPE" {
    # Backend-specific arguments go here.
    # These arguments CANNOT reference variables, locals, or resource attributes.
    # All values must be literals or use partial configuration.
  }
}
```

**Important Rule — No Interpolation in Backend Blocks:**

The following is **INVALID** and will produce an error:

```hcl
# ❌ INVALID: Cannot use variables in a backend block!
terraform {
  backend "s3" {
    bucket = var.state_bucket   # Error: Variables not allowed
    key    = "prod/terraform.tfstate"
    region = var.aws_region     # Error: Variables not allowed
  }
}
```

**Why this restriction exists:** Terraform must configure the backend very early in the initialization process, before it has evaluated any variables, locals, or resources. It's the first thing `terraform init` does. If backend configuration depended on variables, Terraform couldn't know where to fetch state, and therefore couldn't evaluate anything.

---

### 6c.2: Partial Backend Configuration

Since we can't use variables in the backend block, how do we handle dynamic values (like bucket names that differ per environment)? **Partial configuration.**

We leave the dynamic arguments out of the backend block and provide them at `init` time via:

1. A file specified with `-backend-config`
2. Command-line key/value pairs with `-backend-config=KEY=VALUE`

**Method 1: Using a file**

```hcl
# main.tf
# Notice the dynamic arguments are omitted—they will be provided at init time.
terraform {
  backend "s3" {
    # bucket, key, region, etc. NOT specified here
    # We'll provide them when running terraform init
  }
}
```

Create a separate file, `backend.hcl` (by convention named `*.backendname.tfbackend`, e.g., `config.s3.tfbackend`):

```hcl
# backend.hcl — This file contains the actual configuration values.
# It looks just like the contents of the backend block, but without the block wrapper.
# Each line is a simple key = value assignment.

bucket         = "my-state-bucket-prod"
key            = "prod/network/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-locks"
encrypt        = true
```

**Running `init`:**

```bash
terraform init -backend-config=backend.hcl
```

**Explanation of the command:**

- `terraform init` — The initialization command.
- `-backend-config=backend.hcl` — Tells Terraform to read additional backend configuration from the file `backend.hcl`. Terraform merges this with any hardcoded values in the `backend` block (in this case, there are none). This file contains the dynamic environment-specific values.

**Method 2: Using command-line key/value pairs**

```bash
terraform init \
  -backend-config="bucket=my-state-bucket-prod" \
  -backend-config="key=prod/network/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=terraform-locks" \
  -backend-config="encrypt=true"
```

**Using partial configuration makes your code environment-agnostic.** The same `.tf` files can be used for dev, staging, and prod by passing different `backend.hcl` files.

---

### 6c.3: Changing Backends and Migrating State

If you already have a local state file and you want to move to a remote backend, Terraform will detect the change when you run `terraform init` with the new backend configuration.

Terraform will display:
```
Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend
  to the newly configured "s3" backend. No existing state was found in the
  newly configured "s3" backend. Do you want to copy this state to the new
  "s3" backend? Enter "yes" to copy and "no" to start with an empty state.
```

**Key options during backend migration:**

- **Type `yes`:** Terraform copies the local state file to the S3 bucket. This is the correct choice when you're moving an existing project to a remote backend.
- **Type `no`:** Terraform discards the local state and starts fresh with an empty state in S3. This is appropriate if you're starting from scratch.
- **Use `-migrate-state`:** Automatically answers "yes" to the migration prompt. Useful in automation.
- **Use `-reconfigure`:** Tells Terraform to skip the migration check entirely and reconfigure the backend without copying any existing state. This is also used when you're changing backend configuration for the same backend type (e.g., moving to a different S3 bucket).

```bash
# Migrate state from local to S3, automatically answering yes
terraform init -migrate-state -backend-config=backend.hcl

# Reconfigure backend without migration (e.g., changing buckets)
terraform init -reconfigure -backend-config=new-backend.hcl
```

---

### 6c.4: The `cloud` Block vs. `backend` Block

From the offical study guide:

> *You cannot configure a `backend` block when the configuration also contains a `cloud` configuration for storing state data.*

- The `cloud` block is a simplified way to connect directly to HCP Terraform or Terraform Enterprise.
- When you use `cloud`, HCP Terraform manages state storage, locking, and remote execution automatically—no need for a separate `backend` block.
- If you try to use both, `terraform init` will error.

```hcl
# Use THIS for HCP Terraform:
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "my-workspace"
    }
  }
}

# OR use THIS for an independent remote backend (S3, GCS, etc.):
terraform {
  backend "s3" {
    bucket = "my-bucket"
    key    = "state.tfstate"
    region = "us-east-1"
  }
}

# NEVER both in the same configuration!
```

We will cover the `cloud` block in detail in Objective 8.

---

### 6c.5: State File Location After Remote Backend

When you use a remote backend:

- The state file is **no longer stored locally** (except during in-memory operations).
- `terraform.tfstate` will not exist in your working directory (or if it does, it's a leftover from before migration and can be deleted).
- Terraform reads and writes state directly to the remote backend on every operation.

**Exception:** If the remote backend write fails (e.g., network error), Terraform will save a local copy of the state as `errored.tfstate` to prevent data loss. You can use this file to manually push the state with `terraform state push errored.tfstate`.

---

### 6c.6: Hands-On Lab — Simulating a Remote Backend with Local

Since we can't use S3, we'll configure a `local` backend with a custom path to simulate the concept of moving state.

**Directory:** `obj6c-lab`

**Step 1: Create initial config without backend.**

```hcl
# main.tf
resource "local_file" "example" {
  filename = "${path.module}/test.txt"
  content  = "Hello, initial state."
}
```
Run `terraform init && terraform apply -auto-approve`. State is at `terraform.tfstate`.

**Step 2: Add a backend block to simulate migration.**
```hcl
terraform {
  backend "local" {
    path = "remote-state/terraform.tfstate"  # Simulates a remote location
  }
}
```
Run `terraform init`. Terraform will ask: "Do you want to copy existing state to the new backend?" Type `yes`. Terraform moves the state from `./terraform.tfstate` to `./remote-state/terraform.tfstate`.

**Step 3: Verify migration.**
- `terraform.tfstate` no longer exists (or is empty).
- `./remote-state/terraform.tfstate` is the active state.

**Step 4: Observe that `terraform plan` now uses the new state location.**
Any changes to the configuration will be compared against the state in `remote-state/terraform.tfstate`.

This simulates exactly what happens when you migrate from local to S3, just using the local filesystem as the "remote" location.

---

### 6c.7: Mini-Quiz - Configuring Remote State

1. True/False: The `backend` block can reference input variables declared in the same module.
2. Multiple Choice: What is the correct command to migrate an existing local state file to a newly configured S3 backend?
   A. `terraform apply -migrate-state`
   B. `terraform init -migrate-state`
   C. `terraform plan -reconfigure`
   D. `terraform state push`
3. Multiple Answer: Which of the following are valid ways to provide backend configuration values? (Select TWO)
   A. Using variables referenced in the `backend` block.
   B. Using the `-backend-config` command-line flag with a file path.
   C. Using `-backend-config=KEY=VALUE` on the command line.
   D. Setting environment variables named `TF_BACKEND_*`.

<details>

<summary><strong>Show Answers</strong></summary>

1. **False.** The `backend` block cannot reference input variables, locals, or resource attributes. All values in the backend block must be literals or provided via partial configuration at init time.
2. **B.** The correct command to migrate an existing local state file to a newly configured S3 backend is `terraform init -migrate-state`. This command initializes the new backend and automatically answers "yes" to the prompt about copying existing state.
3. **B and C.** Valid ways to provide backend configuration values include using the `-backend-config` command-line flag with a file path (B) and using `-backend-config=KEY=VALUE` on the command line (C). Variables cannot be used directly in the backend block, and there are no environment variables named `TF_BACKEND_*` for backend configuration.

</details>

## Objective 6d: Manage resource drift and Terraform state

*Source: [`Manage resource drift`](https://developer.hashicorp.com/terraform/tutorials/state/resource-drift), [`Use refresh-only mode to sync Terraform state`](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/run/modes-and-options#refresh-only-mode), [`Moved block reference`](https://developer.hashicorp.com/terraform/language/v1.12.x/block/moved), [`Removed block reference`](https://developer.hashicorp.com/terraform/language/v1.12.x/block/removed), [`Refactor Terraform state`](https://developer.hashicorp.com/terraform/language/v1.12.x/state/refactor), [`Terraform State`](https://developer.hashicorp.com/terraform/language/v1.12.x/state)*

### 6d.1: What is Resource Drift?

Drift occurs when the **actual state of your infrastructure** differs from what is recorded in the **Terraform state file** and defined in your **configuration**. Drift happens when someone (or something) makes changes outside of Terraform's control.

**Common causes of drift:**

- A teammate manually changes a security group rule in the AWS Console.
- An auto-scaling event changes the desired capacity of an ASG.
- A resource is deleted accidentally (or maliciously).
- A cloud provider updates a default value for a resource attribute.

### 6d.2: Detecting Drift — Refresh-Only Mode

The recommended approach to detect and review drift is to use the `-refresh-only` flag with `terraform plan`.

```bash
terraform plan -refresh-only
```

**What this does:**

1. Terraform queries the cloud provider APIs for the actual state of all resources.
2. It compares the actual state with the state file.
3. It generates a plan showing what would change in the **state file** (not the infrastructure) to match reality.

**Example output:**
```
# aws_instance.web has been terminated outside of Terraform.
- resource "aws_instance" "web" {
    id = "i-1234567890abcdef0" -> null
  }
```

This plan says: "If I were to update the state file to match reality, I would remove `aws_instance.web` because it no longer exists."

**To accept these state file changes:**
```bash
terraform apply -refresh-only
```

**Important:** This only updates the state file. It does **not** modify your infrastructure. After this, running `terraform plan` will show that Terraform wants to **recreate** the missing instance (because the configuration still defines it, but the state no longer has it).

**Why `-refresh-only` is preferred over the deprecated `terraform refresh`:**
- `terraform refresh` (deprecated) automatically updates the state file **without showing you a plan first**. This can cause surprise state changes.
- `-refresh-only` gives you a preview and requires confirmation, making it much safer.

---

### 6d.3: Resolving Drift

Once drift is detected, you have two options:

1. **Keep the external change:** Update your Terraform configuration to match the new state of the world. Then run `terraform apply` to sync the state.
2. **Revert the external change:** Run `terraform apply` (without `-refresh-only`) so Terraform re-applies your configuration, reverting the drifted resource back to the configured state.

### 6d.4: State Manipulation CLI Commands

Terraform provides commands to inspect and manipulate the state file directly. These are for **advanced use only** and should be avoided in routine workflows.

**`terraform state list`** — Lists all resources in the state file.

```bash
terraform state list
# Output:
# random_pet.server
# local_file.config
# module.vpc.aws_vpc.main
```

**`terraform state show RESOURCE_ADDRESS`** — Shows detailed attributes of a specific resource from the state file.

```bash
terraform state show local_file.config
# Output:
# resource "local_file" "config" {
#     content  = "test"
#     filename = "./config.txt"
#     id       = "abc123..."
# }
```

- **`terraform state rm RESOURCE_ADDRESS`** — Removes a resource from the state file **without destroying the real infrastructure**. Use this when you want Terraform to "forget" about a resource so you can stop managing it without deleting it.

- **`terraform state mv SOURCE DESTINATION`** — Moves a resource from one address to another within the state file. Useful when refactoring and renaming resources.

- **`terraform state pull`** — Downloads the current state from the configured backend and prints it as JSON to stdout. Useful for scripting and inspection.

- **`terraform state push STATE_FILE`** — Uploads a state file to the configured backend. **Extremely dangerous.** Overwrites the remote state. Terraform protects against accidental push by checking lineage and serial numbers, but `-force` bypasses these protections.

---

### 6d.5: The `moved` Configuration Block

Instead of using `terraform state mv` (which is manual and error-prone), Terraform 1.1+ supports the `moved` block to declaratively refactor state.

```hcl
# Suppose we rename our module from 'network' to 'networking'
# We add this moved block so Terraform knows this is a rename, not a destroy/create

moved {
  from = module.network   # The old address in state
  to   = module.networking  # The new address in the configuration
}
```

When you run `terraform plan`, Terraform recognizes that the resource at `module.network` in the state should map to `module.networking` in the configuration. It will move the state entry without destroying or recreating any infrastructure.

**Benefits of `moved` over `state mv`:**

- It's in the configuration, so it's version-controlled and reviewable.
- It's visible during `plan`, so the team can see the refactoring.
- Terraform validates that the move makes sense.

### 6d.6: Importing Existing Resources

To bring existing infrastructure (created outside Terraform) under management, use the `import` block (Terraform 1.5+) or the legacy `terraform import` command.

**Configuration-driven import (Terraform 1.5+):**

```hcl
# Declare the resource you want to import (with its required arguments)
resource "aws_instance" "legacy" {
  instance_type = "t3.micro"
  ami           = "ami-abc123"
}

# The import block tells Terraform to find an existing resource
# with the given ID and bind it to the resource address in state.
import {
  id = "i-1234567890abcdef0"   # The cloud provider's identifier for the resource
  to = aws_instance.legacy     # The Terraform resource address to import into
}
```

Then run `terraform plan -generate-config-out=generated.tf` to optionally generate the full configuration, or just run `terraform apply` to import. Terraform will find the resource in AWS, pull its attributes into the state file, and associate it with `aws_instance.legacy`.

We will cover import in depth in Objective 7.

---

### 6d.7: Hands-On Lab — Drift Detection and State Management

**Directory:** `obj6d-lab`

**File: `main.tf`**

```hcl
# Standard setup
terraform {
  required_providers {
    local = { source = "hashicorp/local", version = "~> 2.5" }
  }
}

# Create a file that we will delete manually to simulate drift
resource "local_file" "config" {
  filename = "${path.module}/config.txt"
  content  = "Important configuration data."
}
```

**Steps:**

1. `terraform init && terraform apply -auto-approve`
2. Verify `config.txt` exists and state is clean.
3. **Simulate drift:** Manually delete `config.txt` (right-click delete in VS Code or `rm config.txt`).
4. Run `terraform plan -refresh-only`. Observe Terraform detects the file is missing and proposes to remove it from state.
5. Run `terraform apply -refresh-only`. Terraform updates the state to reflect the missing file.
6. Run `terraform state list`. The file is gone from state.
7. Run `terraform plan`. Terraform now wants to **create** `config.txt` again because it's in the configuration but not in state after the refresh-only apply.
8. Run `terraform apply`. The file is recreated.

**Additional state exercises:**

- Run `terraform state list` to see the resource address.
- Run `terraform state show local_file.config` to see its attributes.
- Use a `moved` block to rename the resource. Add `moved { from = local_file.config; to = local_file.app_config }` and rename the resource in `main.tf`. Run `terraform plan` to see the move.

---

### 6d.8: State Management Best Practices

Use these habits in real projects:

- Store state in a remote backend with locking for any shared environment.
- Restrict access to state files and backend credentials to the smallest necessary team.
- Prefer `moved`, `import`, `-refresh-only`, and `terraform state` commands over manual edits to `terraform.tfstate`.
- If you need to read outputs from another stack, use `terraform_remote_state` carefully and treat it like an API boundary, not a casual shortcut.
- Before dangerous refactors, take a backup of state and confirm the target backend has versioning or rollback support.
- Never commit state files or backup state files to version control.


---

### 6d.9: Mini-Quiz - Resource Drift and State Management

1. True/False: `terraform apply -refresh-only` modifies both the state file and the real infrastructure to reconcile drift.
2. Multiple Choice: Which command would you use to see all resources currently tracked in the state file?
   A. `terraform show`
   B. `terraform state list`
   C. `terraform plan`
   D. `terraform output`
3. Multiple Answer: Which approaches can be used to safely refactor state when renaming resources? (Select TWO)
   A. `terraform state mv`
   B. The `moved` configuration block
   C. `terraform state rm` followed by `terraform import`
   D. Manually editing `terraform.tfstate`

<details>

<summary><strong>Show Answers</strong></summary>

1. **False.** `terraform apply -refresh-only` only updates the state file to match the actual infrastructure. It does not modify any real resources. To reconcile drift by modifying infrastructure, you would run `terraform apply` without the `-refresh-only` flag after updating the state.
2. **B.** The command to see all resources currently tracked in the state file is `terraform state list`. This lists the addresses of all resources in the state.
3. **A and B.** The `terraform state mv` command can be used to manually move a resource in state, but the `moved` configuration block is the recommended way to declaratively refactor state when renaming resources. Both methods are valid, but the `moved` block provides better visibility and version control. Manually editing `terraform.tfstate` is dangerous and should be avoided.

</details>

## Objective 6: Quiz

> You should know the drill by now, no peeping at the answers until you've given it your best shot!

**Question 1 (True/False):**

The default local backend supports state locking using filesystem advisory locks that are reliable across multiple machines.

**Question 2 (Multiple Choice):**

You have a Terraform project using the local backend. You want to migrate to an S3 backend with DynamoDB locking. You add the `backend "s3"` block to your configuration. What command should you run next?
```
⬜ `terraform apply`
⬜ `terraform init -migrate-state`
⬜ `terraform plan -reconfigure`
⬜ `terraform state push`
```
**Question 3 (Multiple Answer):**

Which of the following are stored directly in the Terraform state file? (Select **THREE**)
```
⬜ The `serial` number
⬜ The `lineage` UUID
⬜ The provider plugin binaries
⬜ Resource attribute values (including secrets)
⬜ The dependency graph of resources
```
**Question 4 (True/False):**

The `backend` block inside `terraform {}` can reference `var.bucket_name` to dynamically set the S3 bucket for state storage.

**Question 5 (Multiple Choice):**

What is the primary reason state locking is essential for team workflows?
```
⬜ It encrypts the state file at rest.
⬜ It prevents two users from running `terraform plan` simultaneously.
⬜ It prevents concurrent `apply` operations from corrupting the state file.
⬜ It enables remote execution of Terraform commands.
```

**Question 6 (Multiple Choice):**

You have an existing local state file with resources. You configure an S3 backend for the first time. After running `terraform init`, Terraform prompts you about migrating state. You type `no`. What happens?
```
⬜ The local state file is encrypted and kept as backup.
⬜ The local state is deleted and the S3 backend starts with an empty state.
⬜ The local state is copied to S3 without any prompt.
⬜ An error is thrown and `init` fails.
```

**Question 7 (True/False):**

`terraform apply -refresh-only` will update your infrastructure to match the configuration if drift is detected.

**Question 8 (Multiple Choice):**

Which CLI command is the **safest** way to inspect what changes would occur to the state file due to external resource modifications?
```
⬜ `terraform refresh`
⬜ `terraform plan -refresh-only`
⬜ `terraform apply -auto-approve`
⬜ `terraform state pull`
```
**Question 9 (Multiple Answer):**

Which backends support state locking? (Select **THREE**)
```
⬜ `local`
⬜ `s3` with a DynamoDB table
⬜ `gcs`
⬜ `azurerm`
⬜ `http`
```

**Question 10 (True/False):**

When using a remote backend, the `terraform.tfstate` file is still written to the local disk after every successful `apply`.

**Question 11 (Multiple Choice):**

What does the `moved` block in Terraform configuration do?
```
⬜ It physically moves cloud resources to a different region.
⬜ It tells Terraform to rename a resource in the state file without destroying the actual resource.
⬜ It backs up the state file to a remote location.
⬜ It locks the state file during long-running operations.
```
**Question 12 (Multiple Choice):**

You accidentally delete an EC2 instance manually via the AWS Console. This instance was managed by Terraform. What will `terraform plan` show the next time you run it?
```
⬜ No changes; Terraform will update the state automatically.
⬜ Terraform will propose to recreate the instance.
⬜ Terraform will error because the state is corrupted.
⬜ Terraform will ask you to import the instance.
```
**Question 13 (True/False):**

The `terraform state rm` command destroys the real-world resource before removing it from the state file.

**Question 14 (Multiple Answer):**

Which of the following are valid methods for providing backend configuration values when they cannot be hardcoded? (Select **TWO**)
```
⬜ Using `-backend-config=KEY=VALUE` at `init` time.
⬜ Using `-backend-config=PATH` to point to a configuration file.
⬜ Setting environment variables like `TF_BACKEND_BUCKET`.
⬜ Using `locals` in the `backend` block.
```

**Question 15 (Multiple Choice):**

What is the effect of running `terraform state rm aws_instance.web`?
```
⬜ Terraform destroys the EC2 instance and removes it from state.
⬜ Terraform removes the EC2 instance from state but does NOT destroy it.
⬜ Terraform locks the state to prevent further changes.
⬜ Terraform renames the resource in state.
```
**Question 16 (True/False):**

The `terraform.tfstate.backup` file is a JSON file that can be manually restored to revert state to the previous version.

**Question 17 (Multiple Choice):**

A configuration has both a `cloud` block and a `backend` block defined in the `terraform {}` block. What happens when you run `terraform init`?
```
⬜ The `cloud` block takes precedence; the `backend` block is ignored.
⬜ The `backend` block takes precedence; the `cloud` block is ignored.
⬜ Terraform merges both configurations.
⬜ Terraform raises an error because they are mutually exclusive.
```
**Question 18 (True/False):**

State locking guarantees that only one `terraform apply` can modify state at a time, but multiple `terraform plan` operations can run concurrently without any issue.

**Question 19 (Multiple Choice):**

You want to move a Terraform-managed resource from `aws_instance.old_name` to `aws_instance.new_name` in the same state file without destroying the instance. Which approach is the most declarative and recommended?
```
⬜ Use `terraform state mv aws_instance.old_name aws_instance.new_name`
⬜ Use a `moved` block: `moved { from = aws_instance.old_name; to = aws_instance.new_name }`
⬜ Edit `terraform.tfstate` directly to change the name
⬜ Run `terraform destroy` and recreate with the new name
```
**Question 20 (Multiple Answer):**

Which of the following statements about the state file are true? (Select **TWO**)
```
⬜ It should be committed to version control to enable collaboration.
⬜ It can contain sensitive data in plaintext.
⬜ It maps configuration resource addresses to real-world resource IDs.
⬜ It is automatically encrypted by Terraform when using the `local` backend.
```

<details>
<summary>Answer</summary>

1. **False.** The local backend uses filesystem advisory locks, which are not reliable across multiple machines. This means that if two users run `terraform apply` at the same time from different machines, they can both read the same state and make changes, but the second one to write will overwrite the first, causing lost changes and corrupted state.
2. **`terraform init -migrate-state`.** This command initializes the new backend and automatically answers "yes" to the prompt about copying existing state from the local backend to the S3 backend. This is the correct way to migrate state when changing backends.
3. **The `serial` number, the `lineage` UUID, and resource attribute values (including secrets) are stored in the state file.** The provider plugin binaries are not stored in the state file; they are downloaded and stored separately on disk. The dependency graph is not explicitly stored in the state file; it is derived from the configuration and resource relationships.
4. **False.** The `backend` block cannot reference variables, locals, or resource attributes. All values in the backend block must be literals or provided via partial configuration at init time. This is because Terraform needs to configure the backend before it can evaluate any variables.
5. **It prevents concurrent `apply` operations from corrupting the state file.** State locking ensures that only one `terraform apply` can modify the state at a time. This prevents two users from making conflicting changes that could corrupt the state file and lead to resource mismanagement.
6. **The local state is deleted and the S3 backend starts with an empty state.** If you type `no` at the migration prompt, Terraform does not copy the existing local state to S3. Instead, it starts with an empty state in the new backend. This means that Terraform will not be aware of any existing resources until you manually import them or recreate them.
7. **False.** `terraform apply -refresh-only` only updates the state file to match the actual infrastructure. It does not modify any real resources. To reconcile drift by modifying infrastructure, you would run `terraform apply` without the `-refresh-only` flag after updating the state.
8. **`terraform plan -refresh-only`.** This command allows you to see what changes would occur to the state file due to external resource modifications without actually applying any changes to the infrastructure. The deprecated `terraform refresh` command would update the state file without showing a plan first, which can lead to surprise changes.
9. **`s3` with a DynamoDB table, `gcs`, and `azurerm` support state locking.** The `local` backend does not support state locking across multiple machines, and the `http` backend does not have built-in locking capabilities.
10. **False.** When using a remote backend, the `terraform.tfstate` file is not written to the local disk. Instead, Terraform reads and writes state directly to the remote backend on every operation. The local `terraform.tfstate` file may exist as a leftover from before migration, but it is not used after the backend is configured.
11. **It tells Terraform to rename a resource in the state file without destroying the actual resource.** The `moved` block allows you to declaratively refactor state when renaming resources. Terraform will move the state entry from the old address to the new address without destroying or recreating any infrastructure.
12. **Terraform will propose to recreate the instance.** Since the instance was deleted outside of Terraform, the next time you run `terraform plan`, Terraform will see that the resource defined in the configuration is missing from the state and will propose to create it again.
13. **False.** The `terraform state rm` command removes a resource from the state file but does NOT destroy the real-world resource. This is useful when you want Terraform to "forget" about a resource so you can stop managing it without deleting it.
14. **Using `-backend-config=KEY=VALUE` at `init` time and using `-backend-config=PATH` to point to a configuration file are valid methods for providing backend configuration values.** Setting environment variables like `TF_BACKEND_BUCKET` is not a standard method for backend configuration, and using `locals` in the `backend` block is not allowed since the backend block cannot reference variables or locals.
15. **Terraform removes the EC2 instance from state but does NOT destroy it.** The `terraform state rm` command only modifies the state file, allowing you to stop managing the resource with Terraform while leaving it intact in the cloud provider.
16. **True.** The `terraform.tfstate.backup` file is a JSON file that contains the previous version of the state before the last successful operation. It can be manually restored by copying it back to `terraform.tfstate` if you need to revert to the previous state.
17. **Terraform raises an error because they are mutually exclusive.** You cannot have both a `cloud` block and a `backend` block in the same configuration. The `cloud` block is used for HCP Terraform or Terraform Enterprise, while the `backend` block is used for independent remote backends. Terraform will not allow both to be configured at the same time.
18. **True.** State locking ensures that only one `terraform apply` can modify the state at a time, preventing corruption. However, multiple `terraform plan` operations can run concurrently without issue since they do not modify state.
19. **Use a `moved` block: `moved { from = aws_instance.old_name; to = aws_instance.new_name }`.** This is the most declarative and recommended approach for refactoring state when renaming resources. It allows you to version control the change and provides visibility during `plan`. Using `terraform state mv` is manual and less transparent, while editing `terraform.tfstate` directly is dangerous and should be avoided.
20. **It can contain sensitive data in plaintext and it maps configuration resource addresses to real-world resource IDs.** The state file can contain sensitive information such as resource attributes, which may include secrets. It also serves as a mapping between the resource addresses defined in your configuration and the actual resource IDs in the cloud provider. The state file should not be committed to version control, and it is not automatically encrypted when using the `local` backend.  

</details>


---

Look at you now, a Terraform state management expert! You've learned about the critical role of the state file, how to manage it with remote backends, detect and resolve drift, and even manipulate state directly when necessary. This knowledge is essential for working with Terraform in real-world scenarios, especially in team environments where collaboration and state integrity are paramount.

Let's proceed to [**Objective 7: Maintain infrastructure with Terraform**](./07-Maintain-infrastructure-with-Terraform.md), where we'll cover how to import existing resources, manage secrets in state, and handle more complex state management scenarios.

