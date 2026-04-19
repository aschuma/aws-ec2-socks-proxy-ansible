# Copilot Workspace Instructions

## What this project does

This repository implements the **AWSSOCKS** workflow: it provisions a temporary AWS EC2 instance that acts as a SOCKSv5 proxy relay, opens an encrypted SSH tunnel to it from the local machine, and tears it all down when finished.

The authoritative functional specification is **`.github/instructions/awssocks-spec.instructions.md`** in this repository.

---

## Technology stack

| Layer | Choice |
|---|---|
| Orchestration | Ansible (`ansible-playbook`) |
| Cloud provider | AWS (EC2, SSM Parameter Store, key pairs, security groups) |
| AWS collection | `amazon.aws` (declared in `collections/requirements.yml`) |
| Target OS | Amazon Linux 2023 (AL2023) |
| Local runtime | Bash wrapper scripts at the repo root |

---

## Repository layout

```
ansible.cfg                  Ansible settings (roles path, inventory)
inventory/hosts.yml          Localhost inventory used by all playbooks
collections/requirements.yml amazon.aws collection declaration

playbooks/
  start.yml                  Provision infra and print the SSH tunnel command
  stop.yml                   Terminate instances and delete cloud resources
  status.yml                 Report current state of all managed resources
  ssh_tunnel_starter.yml     Open the SOCKSv5 tunnel automatically
  config_manager.yml         Interactive config-file selector

roles/
  awssocks_config/           Resolve config from env vars / INI file / defaults
  awssocks_logging/          Timestamped INFO log helper
  awssocks_aws_context/      Validate AWS credentials / session
  awssocks_instance_discovery/ Find managed instances by tag
  awssocks_ami_select/       Select AL2023 AMI via SSM Parameter Store catalog
  awssocks_security_group/   Create / read / delete managed security group
  awssocks_keypair/          Import / read / delete managed SSH key pair
  awssocks_instance/         Launch and terminate managed EC2 instances

START.sh / STOP.sh / STATUS.sh / SSH.sh / CM.sh
                             Thin Bash wrappers that call the matching playbook

.github/
  copilot-instructions.md    ← this file (always-on Copilot context)
  prompts/
    regenerate.prompt.md     Master prompt to regenerate the codebase from scratch
  instructions/
    awssocks-spec.instructions.md  Authoritative functional specification
    ansible.instructions.md       Style rules for playbooks and roles
    configuration.instructions.md Style rules for config loading / INI handling
```

---

## Naming conventions

- **Roles** are named `awssocks_<domain>` (snake_case, always prefixed `awssocks_`).
- **Ansible variables** are prefixed `awssocks_` (lowercase).
- **Environment variables** are prefixed `AWSSOCKS_` (uppercase).
- **AWS resource tags** use `AWSSOCKS__MANAGED = True` (double underscore).
- **Managed security group / key pair** names come from config, never hard-coded.

---

## Configuration precedence (§1.1 of awssocks-spec.instructions.md)

1. Environment variables (`AWSSOCKS_*`) — highest priority
2. `current-config.ini` (symlink managed by the config manager)
3. `config.ini`
4. Built-in defaults

INI files use section `[awssocks]` and lowercase key names that mirror the env-var names.

---

## Key behavioural rules (from awssocks-spec.instructions.md)

- **AMI selection**: query `/aws/service/ami-al2023-latest` SSM path; filter for `al2023` + `gp3`; prefer the configured architecture; fall back gracefully.
- **Security group lookup**: match on the group's *description* field (same value as its name).
- **Instance type**: try candidates in order; skip unavailable types; raise if all fail.
- **Auto-termination**: inject a startup shutdown script only when `AWSSOCKS_AUTO_TERMINATION_AFTER_MINUTES >= 0`.
- **Instance discovery**: tag filter `AWSSOCKS__MANAGED = True`; add state filter `running` when only live instances are needed.
- **Logging**: INFO level only; log resolved config at startup; log every significant action with resource IDs.

---

## User guide (inlined from README.md)

### Requirements

- [uv](https://docs.astral.sh/uv/) — Python package and project manager
- AWS credentials available (for example in `~/.aws/credentials`)
- Local SSH key material (public key at `~/.ssh/<AWSSOCKS_KEY>.pub`)

### One-time setup

After cloning the repository, install the Ansible collections into the project directory:

```bash
uv run ansible-galaxy collection install -r collections/requirements.yml -p ./collections
```

`uv run` automatically creates and syncs the `.venv` from `pyproject.toml` (`ansible-core`, `boto3`, `botocore`) on first use. Collections are installed under `collections/ansible_collections/` (excluded from git). No system-level Ansible, Python packages, or `~/.ansible` writes are required.

### Command playbooks

```bash
# Provision infrastructure and print the SSH tunnel command
ansible-playbook playbooks/start.yml     # or: ./START.sh

# Terminate all managed instances and remove cloud resources
ansible-playbook playbooks/stop.yml      # or: ./STOP.sh

# Report current state without making changes
ansible-playbook playbooks/status.yml    # or: ./STATUS.sh
```

### Helper playbooks

**Configuration Manager** — select the active config file:

```bash
ansible-playbook playbooks/config_manager.yml           # or: ./CM.sh
# Non-interactive:
ansible-playbook playbooks/config_manager.yml -e awssocks_config_selection_index=1
```

**SSH Tunnel Starter** — open the SOCKSv5 tunnel automatically:

```bash
ansible-playbook playbooks/ssh_tunnel_starter.yml       # or: ./SSH.sh
```

---

## What NOT to do

- Do **not** use legacy AL2 SSM path `/aws/service/ami-amazon-linux-latest`.
- Do **not** filter AMI names for `amzn2` or `gp2`.
- Do **not** include `t2.nano` in default instance-type candidates.
- Do **not** hard-code region, key names, or security-group names.
- Do **not** emit DEBUG/TRACE log output in normal operation.
