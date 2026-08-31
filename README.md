# ansible-infra

Ansible automation for managing **Proxmox VE QEMU virtual machines without SSH**.

## Architecture

```text
Semaphore UI
    |
    v
Ansible
    |
    v
Proxmox VE API
    |
    v
QEMU Guest Agent
    |
    v
Virtual machines
```

The Proxmox API is used as the management plane. Commands are executed inside virtual machines through QEMU Guest Agent, so managed VMs do not need to expose SSH or be reachable directly from Semaphore.

## Requirements

- Proxmox VE
- Ansible
- Semaphore UI (optional, but recommended)
- a Proxmox API token
- QEMU Guest Agent installed, running and enabled on managed VMs

Install the required Ansible collections with:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Configuration

The project uses the following environment variables:

| Variable | Description |
| --- | --- |
| `PROXMOX_HOST` | Proxmox hostname or IP |
| `PROXMOX_NODE` | Proxmox node name |
| `PROXMOX_TOKEN_ID` | Proxmox API token ID |
| `PROXMOX_TOKEN_SECRET` | Proxmox API token secret |

Do not commit API secrets to the repository.

## Semaphore UI

Because Ansible communicates with Proxmox rather than directly with the VMs, a simple local inventory is sufficient:

```ini
[local]
localhost ansible_connection=local
```

Store the `PROXMOX_*` variables in a Semaphore Variable Group and attach it to the templates that use this repository.

Some playbooks can optionally accept:

```text
target_vm
```

This allows a specific VM to be selected from Semaphore without modifying the playbook.

## Project structure

```text
ansible-infra/
├── playbooks/        # Proxmox and VM automation
├── ansible.cfg
├── requirements.yml
└── README.md
```

New automation should keep the same principle: **Semaphore → Proxmox API → QEMU Guest Agent**, without introducing direct SSH access to managed VMs.

## Security

The Proxmox API token provides access to the automation layer and must be protected accordingly. Sensitive API calls should use `no_log: true` when appropriate so credentials are not exposed in Ansible or Semaphore output.

QEMU Guest Agent commands may execute with root privileges inside a VM, so the Proxmox API token should only receive the permissions required by the automation.

## Scope

The project currently focuses on QEMU virtual machines. LXC support may be added separately in the future.
