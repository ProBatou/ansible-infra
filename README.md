# ansible-infra

Ansible playbooks for managing **Proxmox VE QEMU virtual machines without SSH**.

The control path is deliberately simple:

```text
Semaphore UI
    |
    v
Ansible on localhost
    |
    v
Proxmox VE API
    |
    v
QEMU Guest Agent
    |
    v
Virtual machine
```

Commands are sent through the Proxmox API to QEMU Guest Agent. Managed VMs therefore do not need to expose SSH to the Ansible/Semaphore host and do not need to exist in an Ansible host inventory.

> [!IMPORTANT]
> This repository currently targets **QEMU virtual machines**. LXC containers are not managed by these playbooks.

## Why this approach?

Traditional Ansible commonly connects directly to every managed host over SSH. This project instead uses Proxmox as the management plane.

That means:

- no SSH credentials for managed VMs;
- no VM IP addresses in the repository;
- no SSH inventory to maintain;
- commands execute as root through QEMU Guest Agent;
- VM discovery is performed directly through the Proxmox API;
- Semaphore only needs network access to the Proxmox API.

## Requirements

### Control side

- Ansible
- Semaphore UI is optional but recommended
- network access to the Proxmox VE API (`8006/tcp`)
- a Proxmox API token with the permissions required for VM discovery and QEMU Guest Agent commands

Install the Ansible collections with:

```bash
ansible-galaxy collection install -r requirements.yml
```

### Proxmox / VM side

For commands executed inside a VM:

1. QEMU Guest Agent must be installed in the guest;
2. the agent service must be running;
3. QEMU Guest Agent support must be enabled for the VM in Proxmox.

The API-only discovery test does not require a working guest agent.

## Configuration

The playbooks read Proxmox credentials from environment variables:

| Variable | Required | Description |
| --- | :---: | --- |
| `PROXMOX_HOST` | yes | Proxmox hostname or IP, without `https://` or port |
| `PROXMOX_NODE` | yes | Proxmox node name |
| `PROXMOX_TOKEN_ID` | yes | API token ID, for example `user@pam!semaphore` |
| `PROXMOX_TOKEN_SECRET` | yes | API token secret |

Never commit the token secret to this repository.

TLS certificate validation is currently disabled in the playbooks to support self-signed/private Proxmox installations. For an Internet-facing or production deployment, trusting the Proxmox CA and enabling certificate validation is preferable.

## Semaphore UI

A minimal Semaphore configuration uses a local inventory because Ansible itself does not connect to the VMs:

```ini
[local]
localhost ansible_connection=local
```

Create a Variable Group containing the four `PROXMOX_*` variables above and attach it to the task templates.

Recommended template settings:

| Setting | Value |
| --- | --- |
| Repository | this repository |
| Inventory | local inventory |
| Variable Group | Proxmox API credentials |

### Selecting a VM

Guest-oriented playbooks accept the optional Ansible variable:

```text
target_vm
```

For example, in a Semaphore Survey you can expose `target_vm` as a string field.

If `target_vm` is omitted, the test/check playbooks select the first available QEMU VM by VMID. This makes initial validation possible without editing the repository.

## Playbooks

### `playbooks/proxmox-test.yml`

Tests authentication and communication with the Proxmox API and lists QEMU virtual machines.

Use this first. It isolates API/token problems from QEMU Guest Agent problems.

### `playbooks/proxmox-agent-test.yml`

Selects a VM and verifies that QEMU Guest Agent responds.

This confirms the path:

```text
Semaphore -> Proxmox API -> QEMU Guest Agent
```

### `playbooks/proxmox-guest-exec-test.yml`

Executes a harmless command inside the selected VM through the Proxmox `agent/exec` endpoint and waits for completion with `exec-status`.

It proves that commands can actually be executed inside the guest without SSH.

### `playbooks/proxmox-apt-check.yml`

For Debian/Ubuntu guests, this playbook:

- checks that `apt-get` is available;
- runs `apt-get update`;
- counts packages with available upgrades;
- reports whether `/var/run/reboot-required` exists;
- does **not** install upgrades.

Example result:

```text
HOSTNAME=my-vm
UPGRADABLE=4
REBOOT_REQUIRED=no
```

## Suggested validation order

When deploying the project on a new Proxmox environment, run:

```text
1. proxmox-test.yml
       |
       v
2. proxmox-agent-test.yml
       |
       v
3. proxmox-guest-exec-test.yml
       |
       v
4. proxmox-apt-check.yml
```

This makes failures easy to locate: API access, Guest Agent availability, command execution, then operating-system maintenance.

## Repository layout

```text
ansible-infra/
├── playbooks/
│   ├── proxmox-test.yml
│   ├── proxmox-agent-test.yml
│   ├── proxmox-guest-exec-test.yml
│   └── proxmox-apt-check.yml
├── .gitignore
├── ansible.cfg
├── requirements.yml
└── README.md
```

## Security model

The Proxmox API token is the sensitive credential in this architecture. Keep it in Semaphore's secret/environment storage or another secrets manager, never in Git.

The current playbooks mark API tasks containing authorization headers with `no_log: true` so token values are not printed in normal Ansible output.

Because QEMU Guest Agent command execution can run commands as root inside guests, grant the API token only the permissions required for this automation and protect access to Semaphore accordingly.

## Current scope

Implemented:

- Proxmox API authentication test;
- QEMU VM discovery;
- optional VM selection by name;
- automatic fallback to the first QEMU VM;
- QEMU Guest Agent health test;
- guest command execution and status polling;
- Debian/Ubuntu APT update check;
- reboot-required detection.

Planned maintenance features can build on the same API/QEMU Guest Agent transport without introducing SSH.

## License

No license has been selected yet. Until a license file is added, normal copyright rules apply.
