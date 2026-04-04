---
title: "Ansible Configured — Six Linux VMs"
date: 2026-04-04
category: infrastructure
outcome: "All six Linux VMs can now be upgraded from a single command on the host machine."
tags: [ansible, automation, linux, virtualbox, infrastructure, ssh]
status: completed
description: "Configured Ansible to manage six VirtualBox Linux VMs — four Oracle Linux and two Debian-based — including SSH access, passwordless sudo, inventory, and an upgrade playbook."
---

## What I did

Completed the Ansible configuration Barrie had started but left unfinished. Created a dedicated SSH key, configured an `ansible` service account on each VM, wrote the inventory and `ansible.cfg`, and built an upgrade playbook that handles both Debian and Oracle Linux automatically.

## The VMs

| VM | OS Family | Hostname |
|----|-----------|----------|
| kali-linux | Debian (Kali) | kali |
| DIS-UBUNTU-SERVER | Debian (Ubuntu) | dis-ubuntu-server |
| ORA-26AI-DB | Oracle Linux | ple-23ai-db |
| PLE-UP-WLS | Oracle Linux | ple-up-wls |
| PLE-UP-OAS | Oracle Linux | ple-up-oas |
| PLE-ORA-DB | Oracle Linux | ple-2012-db |

## What was set up on each VM

- Created `ansible` user with `/bin/bash` shell
- Granted passwordless sudo via `/etc/sudoers.d/ansible`
- Installed SSH public key at `/home/ansible/.ssh/authorized_keys`
- Unlocked account (set password field to `*` for key-only access)
- Enabled SSH service on boot (Oracle Linux: `sshd`; Debian: `ssh`)

## SSH key

Created a new passphrase-free ED25519 key specifically for Ansible:
- `~/.ssh/keys.d/id_ed25519_ansible`

The key has no passphrase so Ansible can use it non-interactively. The existing `id_ed25519` key (which has a passphrase) was not used.

## Files created

- `~/ansible/inventory` — VM list grouped by OS family
- `~/ansible/ansible.cfg` — Ansible configuration (auto-loaded when run from `~/ansible/`)
- `~/ansible/upgrade.yml` — upgrade playbook

## How it works

The upgrade playbook uses `ansible_os_family` to branch automatically:
- Debian/Ubuntu → `apt dist-upgrade` with autoremove
- Oracle Linux → `dnf upgrade`

## Obstacles resolved

- **Kali SSH not running by default** — enabled and started manually; set to auto-start on boot
- **SSH catch-all config offering too many keys** — added a VM-specific `Host` block in `~/.ssh/config` with `IdentitiesOnly yes`
- **Kali PAM two-factor requirement** — added `Match User ansible` / `AuthenticationMethods publickey` to `/etc/ssh/sshd_config` on Kali
- **Account locked after creation without password** — `usermod -p '*'` marks the account as key-only rather than locked
- **Existing SSH key had a passphrase** — created a new dedicated key without one
- **Stale known_hosts entry** — cleared with `ssh-keygen -R ple-23ai-db`
- **Wrong IPs in inventory** — rewrote inventory to use hostnames (resolved via `/etc/hosts`) rather than IPs

## Usage

```bash
# From ~/ansible/

ansible all_linux -m ping                          # Test all VMs
ansible-playbook upgrade.yml                       # Upgrade all six VMs
ansible-playbook upgrade.yml --limit kali          # One VM only
ansible-playbook upgrade.yml --limit oracle_linux  # Oracle Linux group only
ansible-playbook upgrade.yml --limit debian        # Debian group only
```
