# Bitbucket DC Automation

Ansible automation for deploying **Atlassian Bitbucket Data Center** on RHEL 9, including infrastructure installation, PostgreSQL integration, systemd service management, cluster configuration, validation, and uninstallation.

## Repository Structure

```
bitbucket-dc-ansible
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   └── bitbucket.yml          # Variable overrides for the bitbucket group
│   └── hosts.yml                  # Inventory of Bitbucket nodes
├── playbooks
│   ├── install_bitbucket.yml      # Full infrastructure install
│   ├── uninstall_bitbucket.yml    # Removes Bitbucket installation
│   └── validate_bitbucket.yml     # Runs validation tasks only
└── roles
    └── bitbucket
        ├── defaults/main.yml      # Default role variables
        ├── files/                 # Installer archive (atlassian-bitbucket-10.4.2.tar.gz)
        ├── handlers/main.yml      # Restart / reload handlers
        ├── meta/main.yml          # Role metadata
        ├── tasks/                 # Task files (see below)
        ├── templates/             # Jinja2 templates for config files
        └── vars/main.yml          # Role variables
```

## What This Automation Does

The `bitbucket` role is broken into discrete task files, orchestrated by `roles/bitbucket/tasks/main.yml`:

| Task file | Purpose |
|---|---|
| `precheck.yml` | Validates OS (RHEL 9, x86_64 only), verifies the installer archive exists, checks `/app` filesystem disk space, reports existing Java/Git status, warns on HTTP/SSH port conflicts, and tests PostgreSQL connectivity |
| `prerequisites.yml` | Installs required packages (Java 21, Git, PostgreSQL client tooling, etc.), creates the Bitbucket OS user/group, creates home/shared directories, and validates Java 21, Git ≥ 2.42, and `JAVA_HOME` |
| `install.yml` | Extracts the Bitbucket archive, moves it into the target install directory, sets ownership, and validates the start/stop/home-configuration scripts |
| `database.yml` | Validates PostgreSQL connectivity, authentication, server version (17.x), encoding (UTF8), and database ownership |
| `configure.yml` | Creates the shared/log/tmp directories, sets `BITBUCKET_HOME`, and deploys `bitbucket.properties` (JDBC connection, HTTP port, SSH port) |
| `cluster.yml` | Displays Bitbucket Data Center cluster mode information (node, home, shared home) — runs only when `bitbucket_clustered` is `true` |
| `systemd.yml` | Deploys and enables the systemd unit, reloads/restarts on change, and starts the Bitbucket service |
| `validate.yml` | Confirms install/home directories and `bitbucket.properties` exist, verifies the systemd service, waits for the HTTP (and optionally SSH) port, checks the HTTP endpoint, re-validates PostgreSQL connectivity, and confirms the service is active and enabled |
| `uninstall.yml` | Stops/disables the service, removes the systemd unit and installation directory, and optionally purges `BITBUCKET_HOME` and the OS user/group (PostgreSQL is never touched) |

## Playbooks

### `playbooks/install_bitbucket.yml`
Runs the full role (`precheck` → `prerequisites` → `install` → `database` → `configure` → `cluster` [if clustered] → `systemd` → `validate`). Use this for the initial infrastructure setup.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/install_bitbucket.yml
```

### `playbooks/validate_bitbucket.yml`
Runs **only** `validate.yml` against an existing installation. Useful for re-checking service health, HTTP/SSH availability, and PostgreSQL connectivity without re-running the full install.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/validate_bitbucket.yml
```

### `playbooks/uninstall_bitbucket.yml`
Removes the Bitbucket installation via `tasks/uninstall.yml`. By default, `BITBUCKET_HOME` and the PostgreSQL database are preserved; pass `-e bitbucket_purge_data=true` to also remove `BITBUCKET_HOME` and the service account.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/uninstall_bitbucket.yml

# Full purge (also removes BITBUCKET_HOME, OS user/group):
ansible-playbook -i inventory/hosts.yml playbooks/uninstall_bitbucket.yml -e bitbucket_purge_data=true
```

## End-to-End Deployment Flow

1. Populate `inventory/hosts.yml` and `inventory/group_vars/bitbucket.yml` with your environment's values.
2. Place the Bitbucket installer archive (e.g. `atlassian-bitbucket-10.4.2.tar.gz`) in `roles/bitbucket/files/`.
3. Ensure an external PostgreSQL 17 instance is reachable, with the target database/user already created.
4. Run `install_bitbucket.yml` to provision the OS user, install Bitbucket, configure `bitbucket.properties`, and start the service.
5. Open `http://<host>:<bitbucket_http_port>` in a browser and complete any remaining Bitbucket setup steps (license, admin account).
6. (Optional) Run `validate_bitbucket.yml` at any point to re-confirm service health and connectivity.

## Key Variables

These are referenced throughout the role and should be defined in `roles/bitbucket/defaults/main.yml`, `roles/bitbucket/vars/main.yml`, or overridden in `inventory/group_vars/bitbucket.yml`:

| Variable | Description |
|---|---|
| `bitbucket_version` | Bitbucket version being installed |
| `bitbucket_archive` | Filename of the installer `.tar.gz` under `roles/bitbucket/files/` |
| `bitbucket_base_dir` | Base directory used for the disk-space check in precheck |
| `bitbucket_install_dir` | Final Bitbucket installation directory |
| `bitbucket_home` | `BITBUCKET_HOME` directory (shared, log, tmp) |
| `bitbucket_shared_home` | Shared home path (required when `bitbucket_clustered=true`) |
| `bitbucket_user` / `bitbucket_group` | OS user/group Bitbucket runs as |
| `bitbucket_java_package` | Java package installed via `dnf` (OpenJDK 21) |
| `bitbucket_java_home` | Expected `JAVA_HOME` path, validated in prerequisites |
| `bitbucket_http_port` | HTTP port Bitbucket listens on |
| `bitbucket_ssh_port` | SSH port used for Git-over-SSH |
| `bitbucket_service_name` | Name of the systemd service |
| `bitbucket_clustered` | Whether to run cluster configuration tasks |
| `bitbucket_ssh_validation_enabled` | Whether validation waits on the SSH port |
| `bitbucket_validation_url` | URL checked during HTTP validation |
| `bitbucket_validation_retries` / `bitbucket_validation_delay` | Retry/backoff settings for HTTP validation |
| `bitbucket_purge_data` | When `true`, uninstall also removes `BITBUCKET_HOME` and the service account |
| `postgres_host` / `postgres_port` / `postgres_database` / `postgres_user` / `postgres_password` | PostgreSQL connection details |

## Requirements

- **Target OS:** RHEL 9 on x86_64 (enforced by `assert` checks in `precheck.yml`; the playbooks will fail on any other distribution/architecture)
- **Java:** OpenJDK 21 is installed and validated automatically during `prerequisites.yml`
- **Git:** Version ≥ 2.42 is installed and validated automatically during `prerequisites.yml`
- **PostgreSQL:** An external, reachable PostgreSQL **17.x** instance with UTF8 encoding (connectivity, version, encoding, and ownership are checked during `database.yml`)
- **Ansible control node:** `ansible-core` (adjust collection requirements per your Ansible version)
- **Files:** The Bitbucket installer archive (e.g. `atlassian-bitbucket-10.4.2.tar.gz`) must be placed under `roles/bitbucket/files/` before running the install playbook

## Handlers

Defined in `roles/bitbucket/handlers/main.yml`:

- **Reload systemd** — runs `systemd daemon_reload`
- **Restart Bitbucket** — restarts the Bitbucket systemd service (also reloads the daemon)

## Notes

- `database.yml` strictly enforces **PostgreSQL 17.x** and **UTF8** encoding; deployments against other PostgreSQL versions or encodings will fail validation.
- Cluster configuration (`cluster.yml`) currently only displays deployment-mode information (node, home, shared home) and is gated behind `bitbucket_clustered`; it does not yet deploy a `bitbucket.cluster.properties`-style file.
- `uninstall.yml` never removes the PostgreSQL database or role — database lifecycle is intentionally left to a separate PostgreSQL automation (e.g. `postgresql-ansible`).
- By default, uninstall preserves `BITBUCKET_HOME`; pass `bitbucket_purge_data=true` to remove it along with the service user/group.
