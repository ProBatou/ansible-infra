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

The project uses the maintained `community.proxmox` collection instead of implementing Proxmox REST calls manually.

`community.proxmox.proxmox_vm_info` is used for cluster-wide VM discovery and `community.proxmox.proxmox_qemu_api` is the Ansible connection plugin used to execute normal Ansible tasks inside Linux QEMU guests through QEMU Guest Agent. Managed VMs do not need SSH access or direct network connectivity from Semaphore.

## Requirements

- Proxmox VE
- Ansible Core 2.17+
- `community.proxmox` 2.0+
- `proxmoxer` 2.3+
- `requests`
- QEMU Guest Agent installed, enabled and running in managed Linux VMs
- a Proxmox API token with the required Guest Agent permissions

Install the Ansible collection with:

```bash
ansible-galaxy collection install -r requirements.yml
```

The Ansible controller also needs the Python dependencies required by `community.proxmox`, notably `proxmoxer>=2.3.0` and `requests`.

## Configuration

Environment variables:

| Variable | Required | Description |
| --- | --- | --- |
| `PROXMOX_HOST` | yes | Proxmox API hostname or IP for cluster discovery |
| `PROXMOX_TOKEN_ID` | yes | Proxmox API token ID in `user@realm!token` format |
| `PROXMOX_TOKEN_SECRET` | yes | Proxmox API token secret |
| `PROXMOX_VERIFY_SSL` | no | Validate the Proxmox TLS certificate; defaults to `false` in this project |
| `SEMAPHORE_API_TOKEN` | refresh only | Semaphore API token used by the survey refresh playbook |
| `SEMAPHORE_URL` | no | Semaphore base URL; defaults to `http://127.0.0.1:3000` |
| `SEMAPHORE_PROJECT_ID` | no | Semaphore project ID; defaults to `2` |
| `SEMAPHORE_TARGET_TEMPLATE_ID` | no | Template whose Proxmox surveys are refreshed; defaults to `6` |

`PROXMOX_NODE` is no longer required. Nodes are discovered dynamically and selected through the Semaphore Survey.

Secrets must stay in Semaphore/environment configuration and must never be committed to Git.

## Target selection

Guest playbooks use two selectors:

- `target_node`: `all` or a discovered Proxmox node name.
- `target_vms`: one VM, `all`, or `tag:<tag>`.

Examples:

```text
target_node=all
target_vms=tag:Debian

# or

target_node=ProxmoxPB
target_vms=Media
```

The node filter is applied before the VM/tag selector, so the same tag can target a single node or the whole cluster.

Proxmox tags are the preferred way to create dynamic groups without maintaining a separate Ansible inventory.

The reusable selection logic lives in `tasks/proxmox-select-qemu.yml`.

## Semaphore

A minimal local inventory is sufficient for the discovery phase:

```ini
[local]
localhost ansible_connection=local
```

`localhost` is only the Ansible controller used to call the Proxmox and Semaphore APIs. It is not a managed guest VM.

Store the API values in a Semaphore Variable Group.

The playbook `playbooks/proxmox-refresh-semaphore-vm-list.yml` discovers the current cluster state and updates the target template with two Enum surveys:

- **Nœud Proxmox**: `Tout le cluster` plus every discovered node.
- **Cible**: `Toutes les VM`, discovered Proxmox tag groups, then individual VMs with their node shown in the label.

Run the refresh playbook after adding/removing/migrating VMs, changing tags, or changing cluster nodes. It can also be scheduled in Semaphore.

## Security

The Proxmox API token is the automation trust boundary. On Proxmox VE 9+, command execution through the QEMU Guest Agent requires the appropriate `VM.GuestAgent.*` privileges; `VM.GuestAgent.Unrestricted` covers command execution and Guest Agent operations.

The generic command playbook is intended for deliberate one-off administration. Recurrent or sensitive operations should remain dedicated, version-controlled playbooks.

## Scope

The QEMU API connection plugin currently targets Linux QEMU guests. Appliances such as OPNsense or Home Assistant OS should not automatically be assumed compatible with Linux/Debian playbooks. Use Proxmox tags to target appropriate guests.
