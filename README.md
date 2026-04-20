# aws-ec2-socks-proxy-ansible

An Ansible-native implementation of the AWSSOCKS workflow described in [`.github/instructions/awssocks-spec.instructions.md`](https://github.com/aschuma/aws-ec2-socks-proxy-ansible/blob/main/.github/instructions/awssocks-spec.instructions.md).

---

> **Disclaimer:** This project includes content generated with the assistance of artificial intelligence tools. Significant portions of the code, documentation, or other materials may have been created or refined using AI. While efforts have been made to review and validate all outputs, the accuracy and correctness of AI-generated content cannot be guaranteed. Users are encouraged to review and verify the code before use.

---

## Requirements

- [uv](https://docs.astral.sh/uv/) — Python package and project manager
- AWS credentials available (for example in `~/.aws/credentials`)
- Local SSH key material (public key at `~/.ssh/<AWSSOCKS_KEY>.pub`)

### One-time setup

After cloning the repository, install the Ansible collections into the project directory:

```bash
uv run ansible-galaxy collection install -r collections/requirements.yml -p ./collections
```

That is the only manual step. `uv run` automatically creates and syncs the
`.venv` from [pyproject.toml](pyproject.toml) (`ansible-core`, `boto3`,
`botocore`) on first use. Collections are installed under `collections/ansible_collections/`
(excluded from git). No system-level Ansible, Python packages, or `~/.ansible` writes are required.

## Repository Layout

- `ansible.cfg` - Ansible settings
- `inventory/hosts.yml` - Localhost inventory for orchestration
- `playbooks/` - command-oriented and helper playbooks
- `roles/` - reusable implementation units

## Configuration

Configuration precedence:

1. Environment variables (`AWSSOCKS_*`)
2. `current-config.ini` at repo root
3. `config.ini` at repo root
4. Built-in defaults

INI format uses section `[awssocks]` and lowercase keys:

- `awssocks_key`
- `awssocks_region`
- `awssocks_ec2_instance_size`
- `awssocks_ec2_architecture`
- `awssocks_key_name`
- `awssocks_security_group_name`
- `awssocks_auto_termination_after_minutes`
- `awssocks_local_ssh_port`

Defaults:

- `AWSSOCKS_KEY=id_ed25519`
- `AWSSOCKS_REGION=eu-west-2`
- `AWSSOCKS_EC2_INSTANCE_SIZE=t3.nano`
- `AWSSOCKS_EC2_ARCHITECTURE=x86_64`
- `AWSSOCKS_KEY_NAME=AWSSOCKS_KEY`
- `AWSSOCKS_SECURITY_GROUP_NAME=AWSSOCKS_SG`
- `AWSSOCKS_AUTO_TERMINATION_AFTER_MINUTES=-1`
- `AWSSOCKS_LOCAL_SSH_PORT=4444`

## Command Playbooks

### START

```bash
ansible-playbook playbooks/start.yml
```

### STOP

```bash
ansible-playbook playbooks/stop.yml
```

### STATUS

```bash
ansible-playbook playbooks/status.yml
```

## Helper Playbooks

### Configuration Manager

Interactive:

```bash
ansible-playbook playbooks/config_manager.yml
```

Non-interactive (example):

```bash
ansible-playbook playbooks/config_manager.yml \
  -e awssocks_config_selection_index=1
```

### SSH Tunnel Starter

```bash
ansible-playbook playbooks/ssh_tunnel_starter.yml
```

## Roles

- `awssocks_config`: resolve config sources and normalize runtime values
- `awssocks_logging`: timestamped INFO logging
- `awssocks_aws_context`: credential/session validation
- `awssocks_instance_discovery`: discover managed instances
- `awssocks_ami_select`: select AL2023 AMI via SSM catalog
- `awssocks_security_group`: create/read/delete managed security group
- `awssocks_keypair`: import/read/delete managed key pair
- `awssocks_instance`: launch and terminate managed instances
