# JFrog Xray – Ansible Playbook Suite (RHEL 9)

A production-ready Ansible automation suite for installing and upgrading
JFrog Xray on Red Hat Enterprise Linux 9 (and compatible distributions
such as Rocky Linux 9 and AlmaLinux 9).

---

## Project Structure

```
jfrog-xray-ansible/
├── ansible.cfg                          # Project-scoped Ansible configuration
├── site.yml                             # Main entry-point playbook
├── requirements.yml                     # Ansible Galaxy collection dependencies
├── vault/
│   └── secrets.yml                      # Encrypted credentials (Ansible Vault)
├── inventory/
│   ├── hosts.ini                        # Target host definitions
│   └── group_vars/
│       └── xray_servers.yml             # Group-level variable overrides
└── roles/
    └── xray/
        ├── defaults/main.yml            # Default variable values (lowest precedence)
        ├── vars/main.yml                # Internal computed values (higher precedence)
        ├── tasks/
        │   ├── main.yml                 # Orchestration – routes by xray_action
        │   ├── preflight.yml            # Pre-flight checks (hardware, connectivity)
        │   ├── system_prep.yml          # OS packages, user/group, dirs, tuning
        │   ├── install.yml              # Fresh installation workflow
        │   ├── upgrade.yml              # In-place upgrade workflow
        │   ├── configure.yml            # Configuration file management
        │   └── service.yml              # systemd service lifecycle + health checks
        ├── handlers/main.yml            # Event-driven handlers (restart, reload)
        └── templates/
            ├── system.yaml.j2           # JFrog Platform system configuration
            └── xray.config.yaml.j2      # Xray-specific behavioural settings
```

---

## Prerequisites

**Control node (where you run Ansible):**

- Ansible >= 2.14
- Python >= 3.9

**Target nodes (RHEL 9 VMs):**

- RHEL 9.x / Rocky Linux 9.x / AlmaLinux 9.x
- `sudo` access for the service account
- Outbound HTTPS access to `releases.jfrog.io` (or a local mirror)
- An operational JFrog Artifactory instance that Xray will join

**Minimum hardware per Xray node:**

| Resource | Minimum | Recommended (production) |
|----------|---------|--------------------------|
| vCPUs    | 4       | 8+                       |
| RAM      | 8 GB    | 16 GB+                   |
| Disk     | 100 GB  | 500 GB+ (SSD preferred)  |

---

## Quick Start

### 1. Install Galaxy dependencies

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Edit your inventory

Open `inventory/hosts.ini` and replace the example hostnames with your
actual Xray node FQDNs or IP addresses.

### 3. Set your variables

Edit `inventory/group_vars/xray_servers.yml` to set at minimum:

- `xray_version` – the Xray version to install/upgrade to
- `jfrog_platform_url` – your Artifactory base URL
- `xray_db_host` / `xray_db_name` / `xray_db_user` – database connection details

### 4. Encrypt your secrets

```bash
# Edit vault/secrets.yml and fill in real values, then encrypt it:
ansible-vault encrypt vault/secrets.yml

# Add vault references to group_vars/xray_servers.yml:
#   xray_db_password: "{{ vault_xray_db_password }}"
#   jfrog_platform_admin_token: "{{ vault_jfrog_admin_token }}"
```

### 5. Run the playbook

**Fresh installation:**
```bash
ansible-playbook site.yml \
  -e "xray_action=install" \
  --ask-vault-pass
```

**In-place upgrade:**
```bash
ansible-playbook site.yml \
  -e "xray_action=upgrade" \
  -e "xray_version=3.91.0" \
  --ask-vault-pass
```

**Configuration-only (converge drift without touching the binary):**
```bash
ansible-playbook site.yml \
  -e "xray_action=configure" \
  --ask-vault-pass
```

---

## How It Works

### Action Routing

The playbook uses a single `xray_action` variable to select the right
workflow. Think of it like a switchboard: `site.yml` always runs
`preflight` and `system_prep` unconditionally (they are idempotent and
safe to re-run), then branches into either `install.yml` or
`upgrade.yml` based on the action, and finally runs `configure.yml`
and `service.yml` in all cases.

```
site.yml
 └─ roles/xray/tasks/main.yml
       ├─ preflight.yml     (always)
       ├─ system_prep.yml   (always)
       ├─ install.yml       (when xray_action == 'install')
       ├─ upgrade.yml       (when xray_action == 'upgrade')
       ├─ configure.yml     (always)
       └─ service.yml       (always)
```

### Idempotency

Every task uses either a state-aware Ansible module (e.g., `ansible.builtin.file`,
`ansible.builtin.systemd`) or a `creates:` guard on `command:` tasks, so
re-running the playbook against an already-provisioned host converges
configuration drift without re-installing or re-downloading anything unnecessarily.

### Upgrade Safety

The upgrade workflow takes several precautions:

1. **Queue drain** – It polls Xray's `/api/v1/system/ping` endpoint and waits
   for the service to report healthy before stopping it. You can configure
   `xray_upgrade_drain_timeout` (in seconds) to control how long it will wait.
2. **Data backup** – Before touching the binary, it `rsync`s the entire data
   directory to a timestamped folder under `xray_upgrade_backup_dir`. Set
   `xray_upgrade_backup: false` to skip this if you manage snapshots externally.
3. **Version assertion** – After the upgrade script completes, the play
   interrogates the binary and asserts that the reported version matches
   the requested `xray_version`, failing fast if there is a mismatch.

### Variable Precedence

Ansible's variable precedence chain means you can override at any level
without modifying the role itself. From lowest to highest priority:

`roles/xray/defaults/main.yml`  →  `inventory/group_vars/xray_servers.yml`  →
`inventory/host_vars/<host>.yml`  →  `-e` CLI flags

---

## Rolling Upgrades

For a multi-node cluster, you can upgrade one node at a time using the
`xray_serial` variable (maps to Ansible's `serial` directive):

```bash
ansible-playbook site.yml \
  -e "xray_action=upgrade xray_version=3.91.0 xray_serial=1" \
  --ask-vault-pass
```

This ensures at least one node remains available throughout the upgrade window.

---

## Security Notes

- The `vault/secrets.yml` file must **always** be encrypted before committing
  to source control. Add `vault/secrets.yml` to `.gitignore` if you prefer
  to manage it out-of-band.
- The `jfrog_platform_admin_token` should be a scoped Artifactory Access Token
  with only the permissions Xray requires, not a global admin password.
- SELinux and firewalld configuration are enabled by default. Set
  `xray_configure_selinux: false` or `xray_configure_firewall: false` only
  in isolated lab environments.

---

## Troubleshooting

**`assert` fails on CPU/RAM preflight:** The preflight checks use
`ansible_processor_vcpus` and `ansible_memtotal_mb` from gathered facts.
These reflect the VM's physical resources; if your hypervisor is
over-provisioned the checks will reflect the allocated — not available —
resources.

**Download times out:** Set `xray_use_local_mirror: true` and point
`xray_local_mirror_base_url` to a local Artifactory Generic repository that
mirrors `releases.jfrog.io`. This is strongly recommended for air-gapped
environments.

**Service fails to start after install:** Check
`journalctl -u xray -n 100 --no-pager` and `{{ xray_log_dir }}/xray-service.log`
on the target host. The most common causes are a misconfigured `system.yaml`
(wrong DB URL or credentials) or RabbitMQ failing to bind its port.
