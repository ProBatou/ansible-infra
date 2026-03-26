# ansible-infra

Ansible repository for managing Linux VMs in a single production environment using [Ansible](https://www.ansible.com/) and [Semaphore UI](https://www.semui.co/).

## Supported Operating Systems

- **Debian / Ubuntu**
- **RHEL / Rocky Linux**

## Prerequisites

- Ansible ≥ 2.14 installed on the control node
- SSH key-based authentication configured for all managed hosts (no password login)
- The `ansible` user exists on every managed host with `sudo` / `become` privileges
- Python 3 available on managed hosts at `/usr/bin/python3`

## Repository Structure

```
ansible-infra/
├── ansible.cfg                # Ansible configuration (roles_path, inventory, defaults)
├── requirements.yml           # Collection dependencies (community.general >= 7.0.0)
├── inventory/
│   ├── hosts.ini              # Static host groups: [all], [web], [db], [docker_hosts]
│   └── proxmox.yml            # Proxmox dynamic inventory plugin configuration
├── group_vars/
│   ├── all.yml                # Common vars: ansible_user, ntp_servers, base_packages, proxmox_*
│   └── docker_hosts.yml       # Docker-specific vars
├── roles/
│   ├── common/
│   │   ├── handlers/main.yml  # Handlers: restart chrony, sshd
│   │   ├── tasks/main.yml     # Hostname, base packages, NTP, disable root SSH
│   │   └── templates/
│   │       └── chrony.conf.j2 # Chrony NTP configuration template
│   ├── updates/
│   │   └── tasks/main.yml     # apt/dnf full system update + autoremove
│   └── docker/
│       └── tasks/main.yml     # Docker CE + docker-compose-plugin installation
├── playbooks/
│   ├── common.yml             # Apply common role to all hosts
│   ├── updates.yml            # Apply updates role to all hosts
│   └── docker.yml             # Apply docker role to docker_hosts
├── .gitignore
└── README.md
```

## Inventory

Edit `inventory/hosts.ini` to reflect your environment. The default groups are:

| Group          | Purpose                            |
|----------------|------------------------------------|
| `all`          | Every managed host                 |
| `web`          | Web / front-end servers            |
| `db`           | Database servers                   |
| `docker_hosts` | Hosts that run Docker workloads    |

## Running Playbooks

All playbooks use `become: true` (privilege escalation) and require SSH key authentication.

### Apply common configuration (hostname, packages, NTP, SSH hardening)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/common.yml
```

### Apply system updates

```bash
ansible-playbook -i inventory/hosts.ini playbooks/updates.yml
```

### Install Docker

```bash
ansible-playbook -i inventory/hosts.ini playbooks/docker.yml
```

### Limit to a specific host or group

```bash
ansible-playbook -i inventory/hosts.ini playbooks/common.yml --limit web
ansible-playbook -i inventory/hosts.ini playbooks/updates.yml --limit db01
```

### Dry-run (check mode)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/common.yml --check
```

## Vault (secrets)

Sensitive values (passwords, tokens) should be stored in `group_vars/vault.yml` and encrypted with Ansible Vault:

```bash
ansible-vault create group_vars/vault.yml
ansible-vault edit   group_vars/vault.yml
```

Store the vault password in a file named `vault_pass` (excluded from git via `.gitignore`) and use:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/common.yml --vault-password-file vault_pass
```

## Proxmox Dynamic Inventory

This repository ships with a dynamic inventory plugin for Proxmox VE 7.x / 8.x powered by `community.general.proxmox`.

### 1. Install the required collection

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Set the required variables

| Variable           | Description                              |
|--------------------|------------------------------------------|
| `proxmox_url`      | Proxmox API URL, e.g. `https://pve:8006` |
| `proxmox_user`     | Proxmox API user, e.g. `ansible@pam`     |
| `proxmox_password` | Proxmox API password (**secret**)        |

`proxmox_url` and `proxmox_user` can be set in `group_vars/all.yml` or passed with `-e`.  
`proxmox_password` **must** be stored in Semaphore Variable Groups or encrypted with `ansible-vault` — never committed in plain text.

### 3. Test the inventory locally

```bash
ansible-inventory -i inventory/proxmox.yml --list
```

### 4. Configure it in Semaphore UI

1. Go to **Inventory** → **New Inventory**.
2. Set **Type** to **File**.
3. Set the path to `inventory/proxmox.yml`.
4. Pass `proxmox_url`, `proxmox_user`, and `proxmox_password` via a **Variable Group** linked to the template.

## Semaphore UI

1. Add this repository as a **Project** in Semaphore UI.
2. Configure an **Inventory** pointing to `inventory/hosts.ini` (static) or `inventory/proxmox.yml` (dynamic).
3. Create **Task Templates** for each playbook (`common.yml`, `updates.yml`, `docker.yml`).
4. Use **Schedules** for automated recurring updates.

## Tags

Each play is tagged for selective execution:

| Tag       | Scope                         |
|-----------|-------------------------------|
| `common`  | Common configuration tasks    |
| `updates` | System package update tasks   |
| `docker`  | Docker installation tasks     |

```bash
# Run only tasks tagged 'common'
ansible-playbook -i inventory/hosts.ini playbooks/common.yml --tags common
```
