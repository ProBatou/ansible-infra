# ansible-infra

Ansible playbooks designed for Semaphore UI to manage Proxmox VE virtual machines through the Proxmox API and QEMU Guest Agent.

No SSH access to managed VMs is required for the current playbooks.

## Architecture

```text
Semaphore UI
    |
    v
Proxmox API
    |
    v
QEMU Guest Agent
    |
    v
Virtual machine
```

The Ansible controller itself only needs to run the playbooks locally. Commands inside VMs are executed through Proxmox `agent/exec`.

## Requirements

- Proxmox VE with API access
- QEMU Guest Agent installed, enabled and running in VMs that need guest commands
- Ansible / Semaphore UI
- Collections from `requirements.yml`

## Proxmox API variables

Configure these environment variables in Semaphore, preferably through a Variable Group:

| Variable | Description |
| --- | --- |
| `PROXMOX_HOST` | Proxmox host name or IP reachable by Semaphore |
| `PROXMOX_NODE` | Proxmox node name |
| `PROXMOX_TOKEN_ID` | API token ID |
| `PROXMOX_TOKEN_SECRET` | API token secret |
| `PROXMOX_VERIFY_SSL` | Optional. `true` to validate the Proxmox TLS certificate, otherwise defaults to `false` where supported |

Do not commit API secrets to this repository.

## Semaphore inventory

The playbooks run on the Ansible controller itself, so a minimal Semaphore static inventory is enough:

```ini
[local]
localhost ansible_connection=local
```

No inventory of VM IP addresses is required.

## Playbooks

### `playbooks/proxmox-test.yml`

Read-only API connectivity test. Lists QEMU VMs discovered on the configured Proxmox node.

### `playbooks/proxmox-agent-test.yml`

Tests whether QEMU Guest Agent responds on a VM.

Optional extra variable:

```yaml
target_vm: MyVM
```

If `target_vm` is omitted, the playbook selects the first VM by VMID.

### `playbooks/proxmox-guest-exec-test.yml`

Runs a read-only command through QEMU Guest Agent and reports the guest user, hostname and operating system.

Optional extra variable:

```yaml
target_vm: MyVM
```

If omitted, the first VM by VMID is selected.

### `playbooks/proxmox-apt-check.yml`

For Debian/Ubuntu guests, refreshes APT metadata and reports:

- guest hostname
- number of upgradable packages
- whether a reboot is required

It does not install upgrades.

Optional extra variable:

```yaml
target_vm: MyVM
```

If omitted, the first VM by VMID is selected. The selected guest must provide `apt-get` and QEMU Guest Agent.

## Semaphore template example

For an APT check template:

```text
Name: Proxmox - APT Check
Repository: this repository
Playbook: playbooks/proxmox-apt-check.yml
Inventory: local inventory using ansible_connection=local
Variable Group: Proxmox API credentials
```

A Semaphore survey variable named `target_vm` can be added to select the VM when starting the task. It can remain optional if automatic first-VM selection is desired.

## Install Ansible collections

```bash
ansible-galaxy collection install -r requirements.yml
```

Semaphore can also install `requirements.yml` automatically when the repository is cloned for a task.

## Security

The Proxmox API token should only have the permissions required for the operations you intend to expose. QEMU Guest Agent command execution effectively provides administrative command execution inside the selected guest when the guest agent runs with its normal privileges, so protect the Semaphore project, API token and task templates accordingly.
