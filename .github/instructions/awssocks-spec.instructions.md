---
applyTo: "**"
---

# Requirements Document: AWS EC2 SOCKS Proxy Tool

## Overview

This tool provisions and manages a temporary cloud compute instance in a user-selected AWS region that acts as a SOCKSv5 proxy relay. The user connects their local machine to that instance over an encrypted SSH tunnel and configures their browser to route traffic through it. When finished, the user tears down the instance and all associated cloud resources.

---

## 1. Configuration

### 1.1 Configuration Sources and Priority

The tool reads its configuration from two sources, in priority order:

1. **Environment variables** — take precedence over all other sources.
2. **Configuration file** — the tool first looks for a file named `current-config.ini`; if that file is absent, it falls back to `config.ini`.

All configuration values must be resolved before any operation begins. The resolved values must be logged at startup.

### 1.2 Configuration Parameters

| Parameter | Description | Default |
|---|---|---|
| `AWSSOCKS_KEY` | Local filename (without path) of the SSH public key stored in the user's `~/.ssh/` directory | `id_ed25519` |
| `AWSSOCKS_REGION` | AWS region in which to operate | `eu-west-2` |
| `AWSSOCKS_EC2_INSTANCE_SIZE` | Ordered, comma-separated list of instance type candidates | `t3.nano` |
| `AWSSOCKS_EC2_ARCHITECTURE` | Preferred CPU architecture for AMI selection | `x86_64` |
| `AWSSOCKS_KEY_NAME` | Name used to register the public key in AWS | `AWSSOCKS_KEY` |
| `AWSSOCKS_SECURITY_GROUP_NAME` | Name used to create and identify the managed security group in AWS | `AWSSOCKS_SG` |
| `AWSSOCKS_AUTO_TERMINATION_AFTER_MINUTES` | Minutes after launch at which the instance shuts itself down; a negative value disables auto-termination | `-1` |
| `AWSSOCKS_LOCAL_SSH_PORT` | Local port used when forming the SOCKS tunnel command | `4444` |

### 1.3 Configuration File Format

The configuration file uses a standard INI format under a section named `awssocks`. Parameter names inside the file are the lowercase equivalents of the environment variable names listed above.

---

## 2. Core Commands

### 2.1 START

The START command provisions all required cloud infrastructure and outputs the SSH command the user must run to open the proxy tunnel.

**Sequence of operations:**

1. Log the resolved configuration.
2. Query AWS and log all existing instances tagged as managed by this tool (any state).
3. Discover a suitable Amazon Machine Image (AMI) — see §3.
4. Ensure the managed security group exists — see §4.
5. Ensure the managed SSH key pair exists in AWS — see §5.
6. Launch a new compute instance — see §6.
7. Wait until the instance reaches a fully running state.
8. Retrieve and log the instance's public IP address and current state.
9. Print a summary: instance ID, public IP, state.
10. Print the complete SSH command the user must execute to open the SOCKSv5 tunnel.

**SSH command format printed to the user:**

```
ssh -o "StrictHostKeyChecking no" -C -N -i ~/.ssh/<AWSSOCKS_KEY> ec2-user@<PUBLIC_IP> -D <AWSSOCKS_LOCAL_SSH_PORT>
```

### 2.2 STOP

The STOP command terminates all managed instances and removes all associated cloud resources created by this tool.

**Sequence of operations:**

1. Log the resolved configuration.
2. Query AWS and log all existing managed instances (any state).
3. For each managed instance that is not already terminated, terminate it and wait until termination completes.
4. Delete the managed SSH key pair from AWS if it exists.
5. Delete the managed security group from AWS if it exists.

### 2.3 STATUS

The STATUS command reports the current state of all cloud resources managed by this tool without making any changes.

**Sequence of operations:**

1. Log the resolved configuration.
2. Query AWS and log all existing managed instances (any state), including each instance's ID, public IP, and state.
3. Log the ID of the currently installed managed security group, or `None` if absent.
4. Log the name of the currently installed managed SSH key pair, or `None` if absent.

---

## 3. AMI Selection

The tool automatically selects a base machine image using the following rules:

1. Query the cloud provider's public AMI parameter catalog under the path `/aws/service/ami-al2023-latest` (Amazon Linux 2023).
2. Filter to images whose name contains both `al2023` (Amazon Linux 2023) and `gp3` (current-generation general-purpose SSD EBS volume).
3. From that filtered set, prefer images whose name also contains the configured architecture (`AWSSOCKS_EC2_ARCHITECTURE`).
4. Use the first matching image. If no architecture-matching image exists, fall back to any `al2023`/`gp3` image. If none exist at all, use the first image from the full catalog.
5. If the catalog returns no images whatsoever, raise an error and abort.

---

## 4. Security Group Management

### 4.1 Lookup

The tool identifies the managed security group by searching all security groups in the configured region for one whose **description** exactly matches `AWSSOCKS_SECURITY_GROUP_NAME`.

### 4.2 Creation (idempotent)

If no managed security group is found, the tool creates one:

- Name and description are both set to `AWSSOCKS_SECURITY_GROUP_NAME`.
- The group is tagged with `AWSSOCKS__MANAGED = True`.
- An inbound rule is added allowing TCP traffic on port 22 from any source (`0.0.0.0/0`).

If the group already exists, the tool skips creation and returns the existing group's ID.

### 4.3 Deletion

The tool deletes the managed security group by its ID if it exists. If it does not exist, the deletion step is silently skipped.

---

## 5. SSH Key Pair Management

### 5.1 Lookup

The tool identifies the managed key pair by searching all key pairs in the configured region for one whose name exactly matches `AWSSOCKS_KEY_NAME`.

### 5.2 Upload (idempotent)

If no managed key pair is found, the tool reads the public key file located at `~/.ssh/<AWSSOCKS_KEY>.pub` and imports it into AWS under the name `AWSSOCKS_KEY_NAME`.

If the key pair already exists, the tool skips the upload.

### 5.3 Deletion

The tool deletes the managed key pair by name if it exists. If it does not exist, the deletion step is silently skipped.

---

## 6. Instance Lifecycle

### 6.1 Creation

The tool launches a single compute instance with the following properties:

- **Image:** the AMI ID selected per §3.
- **Instance type:** tried in order from the `AWSSOCKS_EC2_INSTANCE_SIZE` candidate list; if a type is unavailable in the region, the next candidate is tried. If all candidates fail, an error is raised.
- **Key pair:** `AWSSOCKS_KEY_NAME` (registered per §5).
- **Security group:** the managed security group ID (configured per §4).
- **Shutdown behavior:** the instance terminates itself (rather than stopping) when shut down.
- **Auto-termination:** if `AWSSOCKS_AUTO_TERMINATION_AFTER_MINUTES` is zero or positive, a startup script is injected that schedules an OS-level shutdown after that many minutes. If the value is negative, no startup script is injected.
- **Tags applied to the instance:**
  - `AWSSOCKS__MANAGED = True`
  - `AWSSOCKS__AUTO_TERMINATION_AFTER_MINUTES = <configured value>`

### 6.2 Instance Discovery

Managed instances are identified by querying for instances with the tag `AWSSOCKS__MANAGED = True` in the configured region.

When only running instances are required (e.g., for the SSH helper), the query additionally filters by `instance-state-name = running`.

### 6.3 Termination

The tool terminates each managed instance by ID and waits until the instance reaches the `terminated` state before proceeding. If the instance is already terminated, the termination step is skipped.

---

## 7. Helper: Configuration Manager

The Configuration Manager is an optional interactive utility for managing multiple named configuration files.

**Behavior:**

1. Reads all `.ini` files from a directory (default: `configs/`).
2. Displays a numbered list of available configuration files, indicating which one is currently active.
3. Prompts the user to select a configuration by number.
4. Creates or updates a symbolic link named `current-config.ini` (default name) pointing to the selected file.
5. The active configuration is identified by the existence and target of this symbolic link.

**Invocation options:**

- `--config-dir <path>` — override the directory containing configuration files.
- `--symlink-path <path>` — override the path of the symlink.

---

## 8. Helper: SSH Tunnel Starter

The SSH Tunnel Starter is an optional utility that automates opening the SOCKSv5 tunnel without requiring the user to manually copy and run the command printed by START.

**Behavior:**

1. Log the resolved configuration.
2. Query AWS for managed instances in the `running` state.
3. If no running instance is found, log a message and exit.
4. If a running instance is found, retrieve its public IP address.
5. Execute the SSH tunnel command (same format as §2.1) as a subprocess, keeping it running indefinitely.
6. When the user sends an interrupt signal (Ctrl-C), gracefully terminate the SSH subprocess, wait for it to exit, and then exit cleanly.

---

## 9. Logging

All operations produce log output. Log entries include:

- The resolved configuration at startup.
- Each significant action (create, delete, wait, complete) with relevant resource identifiers.
- Error conditions with sufficient context to diagnose the failure.

No DEBUG or TRACE logging is emitted in normal operation.

---

## 10. Error Handling

- Any AWS API error that cannot be recovered from (e.g., no AMI found, instance creation failed for all candidate types, VPC missing) causes the operation to abort immediately with an error logged.
- Operations that are no-ops by design (e.g., deleting a resource that does not exist) succeed silently.

---

## 11. External Dependencies and Integrations

| Dependency | Purpose |
|---|---|
| AWS EC2 API | Create/terminate instances, manage key pairs and security groups |
| AWS SSM Parameter Store | Discover the latest Amazon Linux 2023 AMI IDs via the `/aws/service/ami-al2023-latest` parameter path |
| AWS credentials file (`~/.aws/credentials`) | Authenticate all AWS API calls |
| Local SSH public key file (`~/.ssh/<key>.pub`) | Imported into AWS when provisioning the key pair |
| Local `ssh` binary | Invoked as a subprocess by the SSH Tunnel Starter helper |

---

## Appendix: Ambiguities and Gaps

The following items could not be fully resolved from the code alone and may require clarification before re-implementation:

1. **Security group identity by description vs. name.** The security group lookup matches on the group's *description* field rather than its *name*. Both are set to the same value during creation, so this works in practice, but the intent is ambiguous. A re-implementation should decide which field is the canonical identifier.

2. **Multiple running instances.** The SSH Tunnel Starter picks the first running managed instance and ignores any others. The intended behavior when multiple managed instances are running simultaneously is not defined.

3. **Auto-termination labeled BETA.** The auto-termination feature is explicitly marked as a BETA feature in both the configuration template and log output. Its reliability guarantees, failure modes, and expected future state are not documented.

4. **Security group inbound rule scope.** The security group allows SSH from `0.0.0.0/0` (any IPv4 address). There is no allowance for IPv6 (`::/ 0`). Whether this is intentional is not stated.

5. **`configs-template` directory.** The repository includes a template listing suggested configuration filenames (one per AWS region/country), but no mechanism enforces their structure or validates that they exist before use.

6. **No automated tests.** The repository contains no test suite. All behavioral contracts in this document were inferred from source code and README documentation rather than from test specifications.

### Legacy AWS Resource Identifiers

The following AWS identifiers appear in the current source code but are deprecated or superseded. Each is flagged here so that a re-implementation uses current values by default, rather than inheriting them silently.

7. **Legacy SSM parameter path: `/aws/service/ami-amazon-linux-latest`.**
   The existing code queries this path, which serves Amazon Linux 2 (AL2) AMI parameters. Amazon Linux 2 reached end of standard support on **30 June 2025**. New implementations must use the current path `/aws/service/ami-al2023-latest`, which serves Amazon Linux 2023 (AL2023) parameters.

8. **Legacy AMI name filter: `amzn2` (Amazon Linux 2).**
   The existing code filters parameter names containing the string `amzn2`. This identifies Amazon Linux 2 images, which are end-of-life (see item 7). New implementations must filter for `al2023` to target Amazon Linux 2023 images.

9. **Legacy EBS volume type filter: `gp2`.**
   The existing code filters for AMI parameter names containing `gp2`, selecting images backed by the previous-generation General Purpose SSD (`gp2`) EBS volume type. AWS introduced `gp3` in December 2020 as the current-generation replacement, offering better performance at lower cost. New implementations must filter for `gp3` to select `gp3`-backed images. Note: Amazon Linux 2023 AMIs published by AWS default to `gp3` root volumes.

10. **Legacy instance type: `t2.nano`.**
    The existing code's default candidate list includes `t2.nano` as a fallback. The `t2` family is the previous generation of burstable instances and is not available in all regions. The current-generation replacements are `t3.nano` (x86\_64) and `t4g.nano` (ARM/Graviton). New implementations must not include `t2.nano` in the default candidate list.

