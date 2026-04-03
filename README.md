# Automated Server Configuration, Deployment, and Maintenance on GCP

**A complete infrastructure-as-code system for provisioning, configuring, and continuously maintaining a multi-environment server fleet on Google Cloud Platform — driven entirely by Ansible, Bash, and cron from a single management node.**

---

## Table of Contents

- [About The Project](#about-the-project)
- [Technical Architecture](#technical-architecture)
  - [System Overview](#system-overview)
  - [Infrastructure Layout](#infrastructure-layout)
  - [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Core Components](#core-components)
  - [Ansible Playbooks](#ansible-playbooks)
  - [Deployment Script](#deployment-script-deploy-playbookssh)
  - [SSH Activation Script](#ssh-activation-script-addsshsh)
  - [Cron Automation](#cron-automation)
  - [Ansible Inventory](#ansible-inventory-hostsini)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
- [Deployment Workflow](#deployment-workflow)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Documentation](#documentation)

---

## About The Project

Managing server infrastructure manually does not scale. Repeating the same package installs, SSH configurations, and application deployments across multiple servers and environments by hand introduces inconsistency, wastes time, and creates drift between environments that silently breaks things.

This project solves that by treating infrastructure as code. A single Ansible Management Server (AMS) running CentOS acts as the control plane for six Debian target servers organized into three isolated environments — Dev, Test, and Prod. Ansible playbooks handle all provisioning and configuration. A Bash deployment script provides a clean interface for selecting which environment to deploy to. A cron job runs package maintenance automatically, every minute, across all environments, without any manual intervention.

The system supports three distinct use cases:

- **Environment deployment** — provision a full environment (web + hybrid servers) with packages, SSH keys, and the correct application branch in a single command
- **Application updates** — re-run a playbook to pull the latest branch from Git and update the running application
- **Continuous package maintenance** — cron-driven automation ensures all servers stay current without human involvement

---

## Technical Architecture

### System Overview

```
                    ┌──────────────────────────────────┐
                    │     Ansible Management Server    │
                    │     (AMS) — CentOS               │
                    │     Ansible · Bash · Cron        │
                    └──────────────┬───────────────────┘
                                   │
                  ansible-playbook + deploy-playbooks.sh
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
   ┌──────▼──────────┐    ┌────────▼────────┐    ┌──────────▼──────┐
   │   Dev Env       │    │   Test Env      │    │   Prod Env      │
   │─────────────────│    │─────────────────│    │─────────────────│
   │ Web Server      │    │ Web Server      │    │ Web Server      │
   │ Debian          │    │ Debian          │    │ Debian          │
   │ 10.128.0.30     │    │ 10.128.0.26     │    │ 10.128.0.28     │
   │                 │    │                 │    │                 │
   │ Hybrid Server   │    │ Hybrid Server   │    │ Hybrid Server   │
   │ Debian          │    │ Debian          │    │ Debian          │
   │ 10.128.0.31     │    │ 10.128.0.27     │    │ 10.128.0.29     │
   └─────────────────┘    └─────────────────┘    └─────────────────┘

  Web Servers: Apache2 · NodeJS · Git
  Hybrid Servers: Apache2 · NodeJS · Git · MariaDB
```

### Infrastructure Layout

| Environment | Server Type | OS | Internal IP | Installed Services |
|---|---|---|---|---|
| Dev | Web Server | Debian | 10.128.0.30 | Apache2, NodeJS, Git |
| Dev | Hybrid Server | Debian | 10.128.0.31 | Apache2, NodeJS, Git, MariaDB |
| Test | Web Server | Debian | 10.128.0.26 | Apache2, NodeJS, Git |
| Test | Hybrid Server | Debian | 10.128.0.27 | Apache2, NodeJS, Git, MariaDB |
| Prod | Web Server | Debian | 10.128.0.28 | Apache2, NodeJS, Git |
| Prod | Hybrid Server | Debian | 10.128.0.29 | Apache2, NodeJS, Git, MariaDB |
| — | AMS (Control Node) | CentOS | Separate region | Ansible, Bash, Cron |

### Technology Stack

| Category | Technologies |
|---|---|
| **Cloud Platform** | Google Cloud Platform (GCP) |
| **Control Node OS** | CentOS |
| **Target Server OS** | Debian |
| **Orchestration** | Ansible |
| **Playbook Format** | YAML |
| **Scripting** | Bash |
| **Web Server** | Apache2 |
| **Runtime** | NodeJS |
| **Database** | MariaDB |
| **Version Control** | Git, GitHub |
| **Automation** | Cron |
| **Auth** | SSH key pairs, password-based (lab environment) |

---

## Project Structure

```
Google-Cloud-Platform-Automated-Server-Configuration-Deployment-and-Maintenance/
│
├── README.md                                # This file
├── hosts.ini                                # Ansible inventory — host groups, IPs, credentials
│
├── ansible_playbooks/
│   ├── playbook-dev.yml                     # Full provisioning playbook for Dev environment
│   ├── playbook-test.yml                    # Full provisioning playbook for Test environment
│   ├── playbook-prod.yml                    # Full provisioning playbook for Prod environment
│   ├── playbook-packages.yml                # Package-only update playbook for all environments
│   ├── deploy-playbooks.sh                  # Master deployment script (interactive + cron mode)
│   ├── addSSH.sh                            # SSH agent activation script (runs on target servers)
│   └── cronjob.txt                          # Cron job line for automated package maintenance
│
└── Instruction_files/
    ├── Server_Setup_Instructions.txt        # Step-by-step GCP VM setup and initial configuration
    ├── Script_Playbook_Instructions.txt     # Detailed documentation for all scripts and playbooks
    └── common_issues_troubleshooting.txt    # Troubleshooting guide for common setup and runtime issues
```

---

## Core Components

### Ansible Playbooks

The project includes four playbooks. Three are environment-specific full-deployment playbooks; one is a lightweight package-only update playbook used for automated maintenance.

---

#### `playbook-dev.yml` / `playbook-test.yml` / `playbook-prod.yml`

Each environment playbook executes the same five-phase provisioning sequence against its respective host group:

**Phase 1 — Package Installation**

Installs Apache2, NodeJS, and Git on all servers in the environment. Installs MariaDB on hybrid servers only.

```yaml
- name: Install Packages for Dev Environment
  hosts: webserversdev hybridsdev
  tasks:
    - name: Install Apache2
      apt:
        name: apache2
        state: latest
    - name: Install NodeJS
      apt:
        name: nodejs
        state: latest
    - name: Install Git
      apt:
        name: git
        state: latest

- name: Install MariaDB for Hybrid Servers
  hosts: hybridsdev
  tasks:
    - name: Install MariaDB
      apt:
        name: mariadb-server
        state: latest
```

**Phase 2 — SSH Key Configuration**

Reads the AMS's public and private key files and distributes them to target servers. Sets the `.ssh` directory to `0700` and the private key to `0600`.

**Phase 3 — SSH Agent Activation**

Copies `addSSH.sh` to each target server and executes it. This starts the SSH agent on the target and adds the private key, enabling passwordless Git authentication for subsequent operations.

**Phase 4 — GitHub Known Hosts**

Uses `ssh-keyscan` to add `github.com` to each server's known hosts file, preventing interactive fingerprint prompts during automated Git operations.

**Phase 5 — Repository Clone / Update**

Checks whether the application directory exists and whether it contains files, then takes one of three actions:

```yaml
# Clone if directory doesn't exist
- name: Clone Repository
  ansible.builtin.git:
    repo: "{{ repo_url }}"
    dest: "{{ dest }}"
    version: "{{ branch_name }}"
  when: not repo_exists.stat.exists or is_directory_empty.stdout == ""

# Update if directory exists and has content
- name: Update Repository
  ansible.builtin.git:
    repo: "{{ repo_url }}"
    dest: "{{ dest }}"
    version: "{{ branch_name }}"
    update: yes
  when: repo_exists.stat.exists and is_directory_empty.stdout != ""
```

Branch mapping per environment:

| Environment | Playbook | Git Branch |
|---|---|---|
| Dev | `playbook-dev.yml` | `dev` |
| Test | `playbook-test.yml` | `test` |
| Prod | `playbook-prod.yml` | `main` |

---

#### `playbook-packages.yml`

Targets all six servers across all environments simultaneously. Installs and updates Apache2, NodeJS, Git, and MariaDB only — no SSH operations, no Git cloning. Designed for idempotent, non-disruptive execution and is the playbook invoked automatically by cron.

---

### Deployment Script (`deploy-playbooks.sh`)

The master entry point for running any playbook. Supports both interactive and non-interactive (argument-based) execution.

**Interactive mode** — prompts for environment selection:

```bash
./deploy-playbooks.sh

Select the environment:
1) Dev
2) Test
3) Prod
4) Packages for all environments
Enter choice (1/2/3/4):
```

**Non-interactive mode** — pass the choice as an argument (used by cron):

```bash
./deploy-playbooks.sh 4    # Run package updates on all environments
./deploy-playbooks.sh 1    # Deploy to Dev
./deploy-playbooks.sh 3    # Deploy to Prod
```

All playbook runs are logged with `tee` to append to a persistent log file:

```
/home/<username>/logs/deployment.log
```

The script validates that the selected playbook file exists before running and exits with a non-zero code on playbook failure.

---

### SSH Activation Script (`addSSH.sh`)

A minimal two-line script that runs on each target server after the SSH keys have been deployed by the playbook:

```bash
eval "$(ssh-agent -s)"
ssh-add /root/.ssh/id_rsa
```

The playbook copies this script to `/tmp/addSSH.sh` on each target server and executes it as root. This is necessary because SSH agent context is not preserved between Ansible tasks — starting the agent and adding the key in the same script execution ensures the key is available for the subsequent Git clone operation.

---

### Cron Automation

Add the following cron entry on the AMS using `crontab -e`. This runs the package update playbook automatically every minute, every hour, every day, across all environments:

```cron
* * * * * cd /home/<username>/ansible_playbooks && /home/<username>/ansible_playbooks/deploy-playbooks.sh 4
```

Replace `<username>` with the AMS user's home directory name.

Cron output (stdout/stderr) is sent to `/var/spool/mail/<username>` by default. To redirect to a log file instead, append `>> /home/<username>/logs/cron.log 2>&1` to the end of the cron line.

**Why every minute?** The package playbook is idempotent — Ansible only installs or updates packages that are not already at the latest version. Running it frequently ensures the fleet stays current without manual intervention and without risk of double-execution side effects.

---

### Ansible Inventory (`hosts.ini`)

The inventory file organizes hosts into six named groups — one per environment per server type — each with its own connection variables:

```ini
[webserversdev]
10.128.0.30

[webserversdev:vars]
ansible_user=root
ansible_password=<password>

[hybridsdev]
10.128.0.31

[hybridsdev:vars]
ansible_user=root
ansible_password=<password>

# Repeat pattern for webserverstest, hybridstest, webserversprod, hybridsprod
```

This structure allows each playbook to target a specific environment by referencing its host group names, while keeping credentials scoped per group.

> **Note:** See [Security Notes](#security-notes) regarding credential storage in this file.

---

## Getting Started

### Prerequisites

- A Google Cloud Platform account with a project and billing enabled
- Seven GCP Compute Engine VMs provisioned:
  - **1 CentOS VM** — Ansible Management Server (recommended: place in a separate region for resilience)
  - **6 Debian VMs** — 2 per environment (web server + hybrid server)
- SSH root access enabled on all target servers
- Ansible installed on the AMS (`yum install ansible` or equivalent)
- An SSH key pair generated on the AMS (`ssh-keygen -t rsa`)
- A Git repository with `dev`, `test`, and `main` branches for application deployment

### Installation

**1. Clone this repository onto the AMS**

```bash
git clone https://github.com/TDRobertson/Google-Cloud-Platform-Automated-Server-Configuration-Deployment-and-Maintenance.git
cd Google-Cloud-Platform-Automated-Server-Configuration-Deployment-and-Maintenance
```

**2. Configure `hosts.ini`**

Replace the IP addresses with your actual GCP internal IPs. Update `ansible_user` and `ansible_password` to match the root credentials set during VM creation. Copy the completed file to `/etc/ansible/hosts` on the AMS.

```bash
cp hosts.ini /etc/ansible/hosts
```

**3. Place your SSH keys on the AMS**

The playbooks read the AMS's SSH public and private key from `/home/<username>/.ssh/id_rsa.pub` and `/home/<username>/.ssh/id_rsa`. Update the paths in each playbook's `vars` section to match your AMS username.

**4. Make scripts executable**

```bash
chmod +x ansible_playbooks/deploy-playbooks.sh
chmod +x ansible_playbooks/addSSH.sh
```

**5. Run a deployment**

```bash
cd ansible_playbooks
./deploy-playbooks.sh
```

Select an environment when prompted, or pass an argument directly (e.g., `./deploy-playbooks.sh 1` for Dev).

**6. Set up cron for automated package maintenance**

```bash
crontab -e
```

Paste the cron line from `cronjob.txt`, updating `<username>` to match your AMS user.

---

## Deployment Workflow

```
Developer pushes code to GitHub
          │
          ▼
  Run deploy-playbooks.sh
  (interactive or via cron)
          │
          ▼
  Select environment (1/2/3/4)
          │
     ┌────┴────────────────────┐
     │                         │
  Env playbook              Package playbook
  (dev/test/prod)           (all environments)
     │
     ▼
  Phase 1: Install packages (apt)
     │
     ▼
  Phase 2: Distribute SSH keys
     │
     ▼
  Phase 3: Activate SSH agent
     │
     ▼
  Phase 4: Add GitHub to known_hosts
     │
     ▼
  Phase 5: Clone or update repository
  (branch: dev / test / main)
     │
     ▼
  Log results to deployment.log
```

---

## Security Notes

> **This configuration is designed for a controlled lab environment and contains intentional security shortcuts that are not appropriate for production use.**

The following practices in this repository are academic in nature:

- **Root passwords stored in plaintext** — `hosts.ini` stores `ansible_password` values in plain text. In production, use Ansible Vault (`ansible-vault encrypt_string`) to encrypt credentials, or eliminate password-based auth entirely.
- **Root-level SSH access** — All Ansible connections run as `root`. In production, use a dedicated provisioning user with `sudo` and key-only authentication.
- **No credential rotation** — A single shared password is used across all host groups. Production systems should use per-host credentials or certificate-based auth managed by a secrets manager.
- **No network segmentation** — All servers share the same internal IP subnet. Production infrastructure should isolate environments behind separate VPCs or firewall rules.

**For production hardening, replace these patterns with:**
- `ansible_ssh_private_key_file` instead of `ansible_password`
- Ansible Vault for any secrets that must remain in config files
- A non-root provisioning user with scoped `sudo` permissions
- SSH key rotation and `authorized_keys` management via a secrets management system

---

## Troubleshooting

Common issues encountered during setup and playbook execution are documented in `Instruction_files/common_issues_troubleshooting.txt`. Topics covered include:

- YAML indentation errors (spaces only — tabs will break playbook parsing)
- Permission denied errors on target servers
- SSH key and agent configuration failures
- Ansible inventory and connection issues
- Cron job not triggering or logging incorrectly
- Bash script execution permission problems

For a full walkthrough of each playbook's structure and script arguments, see `Instruction_files/Script_Playbook_Instructions.txt`.

---

## Documentation

| File | Purpose |
|---|---|
| `Instruction_files/Server_Setup_Instructions.txt` | Step-by-step guide to provisioning all 7 GCP VMs, configuring SSH, installing Ansible, and running the initial setup |
| `Instruction_files/Script_Playbook_Instructions.txt` | Detailed explanation of every playbook and script — purpose, parameters, expected behavior, and usage examples |
| `Instruction_files/common_issues_troubleshooting.txt` | Troubleshooting guide covering the most common errors encountered during setup and playbook execution |

## Timeline:
This project was originally completed in December 2023, as one of my first academic finals.
