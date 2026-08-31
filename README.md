# ansible-infra

Ansible automation for managing **Proxmox VE QEMU virtual machines without SSH**.

## Architecture

```text
Semaphore UI
    |
    v
Ansible + community.proxmox
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

The project uses the maintained `community.proxmox` collection. `community.proxmox.proxmox_vm_info` provides cluster-wide discovery and `community.proxmox.proxmox_qemu_api` executes normal Ansible tasks through QEMU Guest Agent. Managed VMs do not need SSH access or direct network connectivity from Semaphore.

## Requirements

- Proxmox VE
- Ansible Core 2.17+
- `community.proxmox` 2.0+
- `proxmoxer` 2.3+
- `requests`
- QEMU Guest Agent installed, enabled and running in managed Linux VMs
- a Proxmox API token with the required Guest Agent permissions

Install the collection with `ansible-galaxy collection install -r requirements.yml`. The Ansible controller also needs `proxmoxer>=2.3.0` and `requests`.

## Configuration

| Variable | Required | Description |
| --- | --- | --- |
| `PROXMOX_HOST` | yes | Proxmox API hostname or IP for cluster discovery |
| `PROXMOX_TOKEN_ID` | yes | API token ID in `user@realm!token` format |
| `PROXMOX_TOKEN_SECRET` | yes | API token secret |
| `PROXMOX_VERIFY_SSL` | no | Validate Proxmox TLS certificate; defaults to `false` |
| `SEMAPHORE_API_TOKEN` | refresh only | Semaphore API token used by the survey refresh playbook |
| `SEMAPHORE_URL` | no | Defaults to `http://127.0.0.1:3000` |
| `SEMAPHORE_PROJECT_ID` | no | Defaults to `2` |
| `SEMAPHORE_TARGET_TEMPLATE_ID` | no | Template whose surveys are refreshed; defaults to `6` |

`PROXMOX_NODE` is not required. Nodes are discovered dynamically. Secrets stay in Semaphore/environment configuration and must never be committed to Git.

## Target selection

Guest playbooks use two selectors:

- `target_node`: `all` or a discovered Proxmox node name.
- `target_vms`: `vmid:<id>`, `all`, or `tag:<tag>`.

Examples:

```text
target_node=all
target_vms=tag:Debian

target_node=ProxmoxPB
target_vms=vmid:207
```

Individual VMs are selected by **VMID rather than VM name**. This prevents ambiguity when two cluster nodes contain VMs with the same name. Semaphore still displays a human-readable label such as `Mail [VMID 203] (ProxmoxLF)`, while Ansible receives `vmid:203`.

The node selector can further restrict `all`, a tag group, or an individual VMID. Proxmox tags are the preferred dynamic grouping mechanism. Reusable selection logic lives in `tasks/proxmox-select-qemu.yml`.

## Semaphore

A minimal local inventory is sufficient for discovery:

```ini
[local]
localhost ansible_connection=local
```

`localhost` is only the Ansible controller used to call the Proxmox and Semaphore APIs. It is not a managed guest VM.

The playbook `playbooks/proxmox-refresh-semaphore-vm-list.yml` discovers the current cluster state and updates the target template with two Enum surveys:

- **Nœud Proxmox**: `Tout le cluster` plus every discovered node.
- **Cible**: `Toutes les VM`, discovered Proxmox tag groups, then individual VMs displayed as `Name [VMID id] (Node)`.

Run the refresh playbook after adding/removing/migrating VMs, changing VMIDs/tags, or changing cluster nodes. It can also be scheduled in Semaphore.

## Security

The Proxmox API token is the automation trust boundary. On Proxmox VE 9+, command execution through QEMU Guest Agent requires the appropriate `VM.GuestAgent.*` privileges; `VM.GuestAgent.Unrestricted` covers command execution and Guest Agent operations.

The generic command playbook is intended for deliberate one-off administration. Recurrent or sensitive operations should remain dedicated, version-controlled playbooks.

## Scope

The QEMU API connection plugin currently targets Linux QEMU guests. Appliances such as OPNsense or Home Assistant OS should not automatically be assumed compatible with Linux/Debian playbooks. Use Proxmox tags to target appropriate guests.
