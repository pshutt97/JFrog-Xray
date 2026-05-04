# JFrog Platform Services – Ansible Playbook Suite (RHEL 9)

A production-ready Ansible automation suite for installing and upgrading
**JFrog Xray** and **JFrog Catalog** on Red Hat Enterprise Linux 9 (and
compatible distributions such as Rocky Linux 9 and AlmaLinux 9).

---

## Project Structure

```
jfrog-xray-ansible/
├── ansible.cfg                              # Project-scoped Ansible configuration
├── site.yml                                 # Main entry-point playbook (both services)
├── requirements.yml                         # Ansible Galaxy collection dependencies
├── vault/
│   └── secrets.yml                          # Encrypted credentials (Ansible Vault)
├── inventory/
│   ├── hosts.ini                            # Host definitions (xray_servers + catalog_servers)
│   └── group_vars/
│       ├── xray_servers.yml                 # Xray group-level variable overrides
│       └── catalog_servers.yml              # Catalog group-level variable overrides
└── roles/
    ├── xray/
    │   ├── defaults/main.yml                # Xray default variable values (lowest precedence)
    │   ├── vars/main.yml                    # Xray internal computed values
    │   ├── tasks/
    │   │   ├── main.yml                     # Orchestration – routes by xray_action
    │   │   ├── preflight.yml                # Pre-flight checks (hardware, connectivity)
    │   │   ├── system_prep.yml              # OS packages, user/group, dirs, tuning
    │   │   ├── install.yml                  # Fresh installation workflow
    │   │   ├── upgrade.yml                  # In-place upgrade workflow
    │   │   ├── configure.yml                # Configuration file management
    │   │   └── service.yml                  # systemd service lifecycle + health checks
    │   ├── handlers/main.yml                # Event-driven handlers (restart, reload)
    │   └── templates/
    │       ├── system.yaml.j2               # JFrog Platform system configuration
    │       └── xray.config.yaml.j2          # Xray-specific behavioural settings
    └── catalog/
        ├── defaults/main.yml                # Catalog default variable values
        ├── vars/main.yml                    # Catalog internal computed values
        ├── tasks/
        │   ├── main.yml                     # Orchestration – routes by catalog_action
        │   ├── preflight.yml                # Pre-flight checks
        │   ├── system_prep.yml              # OS packages, user/group, dirs, tuning
        │   ├── install.yml                  # Fresh installation workflow
        │   ├── upgrade.yml                  # In-place upgrade workflow
        │   ├── configure.yml                # Configuration file management
        │   └── service.yml                  # systemd service lifecycle + health checks
        ├── handlers/main.yml                # Event-driven handlers
        └── templates/
            ├── system.yaml.j2               # JFrog Platform system configuration
            └── catalog.config.yaml.j2       # Catalog-specific behavioural settings
```

---

## Design Philosophy

Before diving into usage, it helps to understand two deliberate architectural decisions that shape how the suite works.

**Separate host groups, optional co-location.** Xray and Catalog are managed via two independent Ansible host groups (`xray_servers` and `catalog_servers`) with two independent plays in `site.yml`. This gives you the flexibility to deploy them on dedicated VMs (the default inventory structure) or on the same VMs — just list the same hostnames in both groups. No playbook changes are required; Ansible's idempotent task design means shared preparation work (OS packages, sysctl settings, firewalld) simply reports "ok" on the second pass rather than running twice.

**Structural symmetry between roles.** The `catalog` role is intentionally designed to mirror the `xray` role: same sub-task file names, same workflow ordering, same naming conventions. This means an operator familiar with one role can immediately navigate the other, and skills like "how do I re-run just the configure phase?" transfer directly between both services.

---

## Prerequisites

**Control node:** Ansible 2.14 or higher, and Python 3.9 or higher.

**Target nodes (RHEL 9 VMs):** RHEL 9.x, Rocky Linux 9.x, or AlmaLinux 9.x, with `sudo` access for the connecting service account, outbound HTTPS access to `releases.jfrog.io` (or a configured local mirror), and an operational JFrog Artifactory instance that both services will join.

**Minimum hardware per node:**

| Service | vCPUs (min) | RAM (min) | Disk (min) |
|---------|-------------|-----------|------------|
| Xray    | 4           | 8 GB      | 100 GB     |
| Catalog | 2           | 4 GB      | 50 GB      |

Both services benefit significantly from SSD-backed storage and additional RAM in production. Xray in particular runs an embedded search/index engine whose performance is sensitive to I/O latency.

---

## Quick Start

**Step 1 — Install Galaxy dependencies:**
```bash
ansible-galaxy collection install -r requirements.yml
```

**Step 2 — Edit your inventory.** Open `inventory/hosts.ini` and replace the example hostnames with your actual RHEL 9 node FQDNs or IP addresses. For co-located deployments, list the same hostnames in both `[xray_servers]` and `[catalog_servers]`.

**Step 3 — Set your variables.** Edit `inventory/group_vars/xray_servers.yml` and `inventory/group_vars/catalog_servers.yml` to set at minimum: the target version, the Artifactory platform URL, and the database connection details.

**Step 4 — Encrypt your secrets:**
```bash
# Edit vault/secrets.yml with real values, then encrypt:
ansible-vault encrypt vault/secrets.yml

# Reference vault variables in group_vars files like so:
#   xray_db_password:           "{{ vault_xray_db_password }}"
#   catalog_db_password:        "{{ vault_catalog_db_password }}"
#   jfrog_platform_admin_token: "{{ vault_jfrog_admin_token }}"
```

**Step 5 — Run the playbook:**

Install both services in a single run:
```bash
ansible-playbook site.yml \
  -e "xray_action=install catalog_action=install" \
  --ask-vault-pass
```

Upgrade Xray only:
```bash
ansible-playbook site.yml --tags xray \
  -e "xray_action=upgrade xray_version=3.91.0" \
  --ask-vault-pass
```

Upgrade Catalog only:
```bash
ansible-playbook site.yml --tags catalog \
  -e "catalog_action=upgrade catalog_version=1.6.0" \
  --ask-vault-pass
```

Upgrade both services simultaneously:
```bash
ansible-playbook site.yml \
  -e "xray_action=upgrade    xray_version=3.91.0" \
  -e "catalog_action=upgrade catalog_version=1.6.0" \
  --ask-vault-pass
```

Converge configuration drift without touching the binary:
```bash
ansible-playbook site.yml \
  -e "xray_action=configure catalog_action=configure" \
  --ask-vault-pass
```

---

## How the Action Routing Works

Think of `xray_action` and `catalog_action` as a switchboard at the top of each role's task orchestration. Every play always runs `preflight` and `system_prep` unconditionally — those phases are safe and idempotent — then branches into either `install.yml` or `upgrade.yml` depending on the action variable. Finally, `configure.yml` and `service.yml` run in all cases to ensure configuration is reconciled and the service is healthy regardless of which branch was taken.

```
site.yml
 ├─ Play 1: xray_servers  → roles/xray/tasks/main.yml
 │     ├─ preflight.yml     (always)
 │     ├─ system_prep.yml   (always)
 │     ├─ install.yml       (when xray_action == 'install')
 │     ├─ upgrade.yml       (when xray_action == 'upgrade')
 │     ├─ configure.yml     (always)
 │     └─ service.yml       (always)
 └─ Play 2: catalog_servers → roles/catalog/tasks/main.yml
       ├─ preflight.yml     (always)
       ├─ system_prep.yml   (always)
       ├─ install.yml       (when catalog_action == 'install')
       ├─ upgrade.yml       (when catalog_action == 'upgrade')
       ├─ configure.yml     (always)
       └─ service.yml       (always)
```

The preflight tasks enforce mutual exclusivity — if a service is already installed you cannot run the install action against it, and vice versa — preventing the most common class of accidental double-installs in production.

---

## Variable Precedence

Ansible resolves variables from lowest to highest precedence: `roles/*/defaults/main.yml` → `inventory/group_vars/*.yml` → `inventory/host_vars/<host>.yml` → `-e` CLI flags. The practical implication is that `defaults/main.yml` is safe to read as documentation — it describes every variable the role supports with sensible fallbacks. Operator customisation belongs in `group_vars` or `host_vars`, and one-time overrides (like a specific version for an upgrade run) go on the CLI.

---

## Rolling Upgrades

For multi-node deployments, pass `xray_serial=1` or `catalog_serial=1` to process one node at a time, keeping the rest of the cluster available throughout the maintenance window:

```bash
ansible-playbook site.yml --tags xray \
  -e "xray_action=upgrade xray_version=3.91.0 xray_serial=1" \
  --ask-vault-pass
```

---

## Co-locating Xray and Catalog

When both services run on the same VMs, list those hosts in both inventory groups and ensure there are no port conflicts. The default ports are:

| Service | HTTP | gRPC | Router |
|---------|------|------|--------|
| Xray    | 8000 | 7777 | 8082   |
| Catalog | 8082 | 7788 | 8049   |

Note that Xray's default router port (8082) conflicts with Catalog's default HTTP port. In a co-located deployment, override one of them in the appropriate `host_vars` file — for example, set `catalog_http_port: 8083`. Both services also require their own PostgreSQL database (`xraydb` and `catalogdb` by default); they can share the same PostgreSQL server but must use separate databases and credentials.

---

## Upgrade Safety

Both upgrade workflows follow the same conservative sequence. They poll the service's health endpoint before stopping it (giving in-flight operations a chance to finish), then optionally rsync the data directory to a timestamped backup before touching any binaries. Set `xray_upgrade_backup: false` or `catalog_upgrade_backup: false` if you prefer to handle backups at the hypervisor or storage layer. After the upgrade script runs, the play starts the service and hard-fails if the reported binary version does not match the requested version, so you know immediately if something went wrong rather than discovering it later in monitoring.

---

## Security Notes

The `vault/secrets.yml` file must always be encrypted before committing to source control. Adding it to `.gitignore` as a defence-in-depth measure is also strongly recommended. The `jfrog_platform_admin_token` shared by both services should be a scoped Artifactory Access Token, not a global admin password. SELinux and firewalld configuration are both enabled by default; disable them only in isolated lab environments.

---

## Troubleshooting

**Port conflict on a co-located host.** If `preflight.yml` fails the port check during an install, run `ss -tlnp | grep <port>` on the target host to identify which process owns the port and adjust the relevant port variable in `host_vars`.

**Download timeouts.** Set `xray_use_local_mirror: true` or `catalog_use_local_mirror: true` and configure the corresponding `*_local_mirror_base_url` to point at an Artifactory Generic repository that mirrors `releases.jfrog.io`. This is essential for air-gapped environments and significantly faster in general.

**Service fails to start after install.** Check `journalctl -u xray -n 100 --no-pager` or `journalctl -u catalog -n 100 --no-pager` on the target host. The most common causes are a misconfigured `system.yaml` (wrong database URL or credentials) or a failure to join the JFrog Platform (wrong Artifactory URL or expired admin token).

**Version assertion fails after upgrade.** This indicates the upgrade script ran but the binary was not replaced. Check the upgrade script's stdout output (logged by the `Display upgrade script output` task) for error messages, and inspect the install directory to confirm which binary version is present.
