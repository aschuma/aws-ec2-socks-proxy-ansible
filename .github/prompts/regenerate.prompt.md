---
mode: agent
description: >
  Regenerate the entire AWSSOCKS codebase from scratch using the requirements
  in awssocks-spec.instructions.md. Run this prompt when you want to rebuild with the current tech
  stack, swap to a different stack, or apply large-scale changes.
---

# Regenerate AWSSOCKS Codebase

## Requirements source

All functional requirements are defined in [`.github/instructions/awssocks-spec.instructions.md`](../instructions/awssocks-spec.instructions.md).
Read that file in full before generating any code. Every behavioural detail —
configuration precedence, AMI selection rules, security group management,
instance lifecycle, logging, error handling — is specified there. Do not invent
behaviour that is not in `awssocks-spec.instructions.md`.

## Technology

**Technology: Ansible**

Implement the project using Ansible playbooks and roles as described below.
To switch stacks, change the "Technology" line above (e.g. to `Terraform` or
`Python`) and update the "Target layout" section accordingly before running
this prompt.

## Target layout

Generate or regenerate **all** of the following, replacing any existing
content:

```
ansible.cfg
inventory/hosts.yml
collections/requirements.yml
group_vars/all.yml

playbooks/start.yml
playbooks/stop.yml
playbooks/status.yml
playbooks/ssh_tunnel_starter.yml
playbooks/config_manager.yml

roles/awssocks_config/
roles/awssocks_logging/
roles/awssocks_aws_context/
roles/awssocks_instance_discovery/
roles/awssocks_ami_select/
roles/awssocks_security_group/
roles/awssocks_keypair/
roles/awssocks_instance/

START.sh
STOP.sh
STATUS.sh
SSH.sh
CM.sh
```

Do **not** modify:
- `.github/` (this directory)
- `pyproject.toml` and `uv.lock` (Python environment managed by uv)
- `current-config.ini` (user-managed symlink)
- `configs/` (user-managed configuration files)

The `README.md` at the repository root is the user-facing documentation.
Its current content (inlined for reference) is:

```
# aws-ec2-socks-proxy-ansible

An Ansible-native implementation of the AWSSOCKS workflow described in
`.github/instructions/awssocks-spec.instructions.md`.

---

> **Disclaimer:** This project includes content generated with the assistance
> of artificial intelligence tools. Significant portions of the code,
> documentation, or other materials may have been created or refined using AI.
> While efforts have been made to review and validate all outputs, the accuracy
> and correctness of AI-generated content cannot be guaranteed. Users are
> encouraged to review and verify the code before use.

---

## Requirements

- uv — Python package and project manager
- AWS credentials available (for example in ~/.aws/credentials)
- Local SSH key material (public key at ~/.ssh/<AWSSOCKS_KEY>.pub)

### One-time setup

After cloning the repository, install the Ansible collections into the project directory:

  uv run ansible-galaxy collection install -r collections/requirements.yml -p ./collections

uv run automatically creates and syncs the .venv from pyproject.toml
(ansible-core, boto3, botocore) on first use. Collections are installed
under collections/ansible_collections/ (excluded from git). No system-level
Ansible, Python packages, or ~/.ansible writes are required.

## Repository Layout

- ansible.cfg — Ansible settings
- inventory/hosts.yml — Localhost inventory for orchestration
- group_vars/all.yml — Shared variables available to all plays
- playbooks/ — command-oriented and helper playbooks
- roles/ — reusable implementation units
- configs/ — named .ini configuration files (one per target region/profile)
- current-config.ini — symlink to the active config, managed by CM.sh

## Configuration

Configuration precedence:

1. Environment variables (AWSSOCKS_*)
2. current-config.ini at repo root
3. config.ini at repo root
4. Built-in defaults

INI format uses section [awssocks] and lowercase keys:
  awssocks_key, awssocks_region, awssocks_ec2_instance_size,
  awssocks_ec2_architecture, awssocks_key_name,
  awssocks_security_group_name, awssocks_auto_termination_after_minutes,
  awssocks_local_ssh_port

Defaults:
  AWSSOCKS_KEY=id_ed25519, AWSSOCKS_REGION=eu-west-2,
  AWSSOCKS_EC2_INSTANCE_SIZE=t3.nano, AWSSOCKS_EC2_ARCHITECTURE=x86_64,
  AWSSOCKS_KEY_NAME=AWSSOCKS_KEY, AWSSOCKS_SECURITY_GROUP_NAME=AWSSOCKS_SG,
  AWSSOCKS_AUTO_TERMINATION_AFTER_MINUTES=-1, AWSSOCKS_LOCAL_SSH_PORT=4444

## Command Playbooks

  ansible-playbook playbooks/start.yml   # or: ./START.sh
  ansible-playbook playbooks/stop.yml    # or: ./STOP.sh
  ansible-playbook playbooks/status.yml  # or: ./STATUS.sh

## Helper Playbooks

Configuration Manager (interactive):
  ansible-playbook playbooks/config_manager.yml           # or: ./CM.sh
Non-interactive:
  ansible-playbook playbooks/config_manager.yml -e awssocks_config_selection_index=1

SSH Tunnel Starter:
  ansible-playbook playbooks/ssh_tunnel_starter.yml       # or: ./SSH.sh

## Roles

- awssocks_config: resolve config sources and normalize runtime values
- awssocks_logging: timestamped INFO logging
- awssocks_aws_context: credential/session validation
- awssocks_instance_discovery: discover managed instances
- awssocks_ami_select: select AL2023 AMI via SSM catalog
- awssocks_security_group: create/read/delete managed security group
- awssocks_keypair: import/read/delete managed key pair
- awssocks_instance: launch and terminate managed instances
```

Regeneration must keep `README.md` consistent with the above structure.

## Ansible conventions to follow

- Role names: `awssocks_<domain>` (snake_case, `awssocks_` prefix mandatory).
- All Ansible variables: `awssocks_` prefix, lowercase.
- All environment variables: `AWSSOCKS_` prefix, uppercase.
- AWS resource tags: `AWSSOCKS__MANAGED: "True"` (double underscore).
- Use `amazon.aws` collection modules for all AWS API calls.
- Every role must have `tasks/main.yml`; add `defaults/main.yml` for role-local defaults.
- Playbooks import roles; they do not contain task logic directly.
- `ansible.cfg` must set `roles_path`, `inventory`, and disable host-key checking for localhost.

## Configuration implementation rules (awssocks-spec.instructions.md §1)

- Read config in order: env vars → `current-config.ini` → `config.ini` → defaults.
- Resolve all values before any operation and log them at INFO level.
- INI section name: `awssocks`; key names are lowercase equivalents of env-var names.
- Default values (from awssocks-spec.instructions.md §1.2):
  - `AWSSOCKS_KEY=id_ed25519`
  - `AWSSOCKS_REGION=eu-west-2`
  - `AWSSOCKS_EC2_INSTANCE_SIZE=t3.nano`
  - `AWSSOCKS_EC2_ARCHITECTURE=x86_64`
  - `AWSSOCKS_KEY_NAME=AWSSOCKS_KEY`
  - `AWSSOCKS_SECURITY_GROUP_NAME=AWSSOCKS_SG`
  - `AWSSOCKS_AUTO_TERMINATION_AFTER_MINUTES=-1`
  - `AWSSOCKS_LOCAL_SSH_PORT=4444`

## AMI selection rules (awssocks-spec.instructions.md §3)

- Query SSM path `/aws/service/ami-al2023-latest` (AL2023 — **not** AL2).
- Filter parameter names containing both `al2023` and `gp3`.
- Prefer images also matching the configured architecture.
- Fall back as described in §3 of awssocks-spec.instructions.md.
- **Never** use `/aws/service/ami-amazon-linux-latest`, `amzn2`, `gp2`, or `t2.nano`.

## Bash wrapper scripts

Each script (`START.sh`, etc.) must:
- Use `#!/usr/bin/env bash` and `set -euo pipefail`.
- Resolve its own directory via `BASH_SOURCE[0]` and `cd` to the repo root.
- Pass all CLI arguments through to `ansible-playbook` with `"$@"`.
- Be executable (`chmod +x`).

## Logging rules (awssocks-spec.instructions.md §9)

- All output at INFO level with timestamps.
- Log resolved config at startup.
- Log every significant action (create, delete, wait, complete) with resource IDs.
- No DEBUG or TRACE output in normal operation.

## Error handling rules (awssocks-spec.instructions.md §10)

- Unrecoverable AWS API errors abort the operation immediately.
- No-op operations (deleting a resource that does not exist) succeed silently.

## Deliverable

After generating all files, print a brief summary listing:
1. Files created or replaced.
2. Any ambiguities encountered (refer to awssocks-spec.instructions.md Appendix for known gaps).
3. Any assumptions made that are not explicitly stated in awssocks-spec.instructions.md.
