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

`community.proxmox.proxmox_vm_info` is used for VM discovery and `community.proxmox.proxmox_qemu_api` is the Ansible connection plugin used to execute normal Ansible tasks inside Linux QEMU guests through QEMU Guest Agent. Managed VMs do not need SSH access or direct network connectivity from Semaphore.

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
| `PROXMOX_HOST` | yes | Proxmox API hostname or IP |
| `PROXMOX_TOKEN_ID` | yes | Proxmox API token ID |
| `PROXMOX_TOKEN_SECRET` | yes | Proxmox API token secret |
| `PROXMOX_NODE` | no | Restrict discovery to one Proxmox node |
| `PROXMOX_VERIFY_SSL` | no | Validate the Proxmox TLS certificate; defaults to `false` in this project |

Secrets must stay in Semaphore/environment configuration and must never be committed to Git.

## VM selection

Playbooks that act on guests use the common `target_vms` selector.

Supported forms:

```text
Media
aVM,anotherVM
all
tag:debian
tag:infra
```

Proxmox tags are therefore the preferred way to create dynamic groups without maintaining a separate Ansible inventory.

The reusable selection logic lives in `tasks/proxmox-select-qemu.yml`.

## Semaphore

A minimal local inventory is sufficient for the discovery phase:

```ini
[local]
localhost ansible_connection=local
```

Store the `PROXMOX_*` values in a Semaphore Variable Group.

For playbooks using VM selection, expose `target_vms` as a Survey variable. Commands or other playbook-specific parameters can be exposed as additional Survey variables.

## Security

The Proxmox API token is the automation trust boundary. On Proxmox VE 9+, command execution through the QEMU Guest Agent requires the appropriate `VM.GuestAgent.*` privileges; `VM.GuestAgent.Unrestricted` covers command execution and Guest Agent operations.

The generic command playbook is intended for deliberate one-off administration. Recurrent or sensitive operations should remain dedicated, version-controlled playbooks.

## Scope

The QEMU API connection plugin currently targets Linux QEMU guests. Appliances such as OPNsense or Home Assistant OS should not automatically be assumed compatible with Linux/Debian playbooks. Use Proxmox tags to target appropriate guests.
