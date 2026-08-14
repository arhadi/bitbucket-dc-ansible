# Bitbucket Data Center Ansible Automation

Production-oriented Ansible automation for installing, configuring, validating, and uninstalling Atlassian Bitbucket Data Center on RHEL 9 with PostgreSQL.

The role supports **two installation methods**:

- **Archive installation** using the Atlassian `.tar.gz` distribution.
- **Linux binary installer** using the Atlassian `-x64.bin` installer in unattended Install4j mode.

Both methods converge on the same Ansible-managed configuration, systemd service, validation, and uninstall lifecycle.

> Current tested Bitbucket version: **10.4.2**

## Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Validated Reference Configuration](#validated-reference-configuration)
- [Directory Structure](#directory-structure)
- [Requirements](#requirements)
- [Installation Methods](#installation-methods)
- [Preparing the Installers](#preparing-the-installers)
- [Inventory](#inventory)
- [Configuration Variables](#configuration-variables)
- [Usage](#usage)
- [Role Task Flow](#role-task-flow)
- [Binary Installer Response File](#binary-installer-response-file)
- [Database Configuration](#database-configuration)
- [Bitbucket Home and Shared Home](#bitbucket-home-and-shared-home)
- [Systemd Ownership Model](#systemd-ownership-model)
- [Data Center / Cluster Mode](#data-center--cluster-mode)
- [Validation](#validation)
- [Uninstall Behavior](#uninstall-behavior)
- [Reinstallation](#reinstallation)
- [Troubleshooting](#troubleshooting)
- [Security Recommendations](#security-recommendations)
- [Tested Lifecycle](#tested-lifecycle)
- [Production Readiness Notes](#production-readiness-notes)
- [Summary](#summary)

## Overview

The `bitbucket` role manages the Bitbucket Data Center application lifecycle:

```text
precheck
   |
   v
prerequisites
   |
   v
installation method dispatcher
   |
   +-- archive   -> install_archive.yml
   |
   +-- installer -> install_installer.yml
   |
   v
database
   |
   v
configure
   |
   v
cluster
   |
   v
systemd
   |
   v
validate
```

The installation method is selected using:

```yaml
bitbucket_install_method: "installer"
```

or:

```yaml
bitbucket_install_method: "archive"
```

The Linux binary installer path was successfully exercised with Bitbucket Data Center 10.4.2.

The automation deliberately keeps application lifecycle management under Ansible. The Atlassian binary installer installs the product files but does not create or start its own service. Ansible subsequently applies the desired configuration and manages Bitbucket through `bitbucket.service`.

## Key Features

- RHEL 9 pre-installation validation.
- Bitbucket Data Center 10.4.2.
- Dual installation methods: `.bin` and `.tar.gz`.
- SHA256 validation of selected installation media.
- Custom `/app` installation layout.
- Dedicated `bitbucket` OS user and group.
- PostgreSQL connectivity validation.
- PostgreSQL database preserved during uninstall.
- Bitbucket HTTP port configuration.
- Bitbucket SSH port configuration.
- JVM heap configuration.
- G1GC runtime configuration.
- `BITBUCKET_HOME` management.
- Bitbucket shared directory management.
- Data Center deployment support.
- Ansible-managed `bitbucket.properties`.
- Ansible-managed systemd service.
- Application process validation.
- HTTP validation.
- Optional SSH validation.
- Separate validation playbook.
- Safe uninstall with application-data preservation by default.
- Installer and archive workflows converge on a common configuration lifecycle.

## Validated Reference Configuration

The current automation was tested with the following configuration:

| Component | Tested value |
|---|---|
| Product | Atlassian Bitbucket Data Center |
| Version | `10.4.2` |
| OS | RHEL `9.8` |
| Architecture | `x86_64` |
| Preferred install method | Linux binary installer |
| Binary installer | `atlassian-bitbucket-10.4.2-x64.bin` |
| Binary SHA256 | `4b3f5026cf3afd1a60d0153840dc377433e8a3a6b358bfb258e43e9c3b2c0d27` |
| Archive | `atlassian-bitbucket-10.4.2.tar.gz` |
| Install directory | `/app/bitbucket-10.4.2` |
| BITBUCKET_HOME | `/app/bitbucket-data` |
| Shared directory | `/app/bitbucket-data/shared` |
| OS user/group | `bitbucket:bitbucket` |
| HTTP port | `7990` |
| SSH port | `7999` |
| PostgreSQL host | `127.0.0.1` |
| PostgreSQL port | `15432` |
| Database | `bitbucket` |
| Database user | `bitbucket` |
| JVM minimum heap | `2g` |
| JVM maximum heap | `4g` |
| systemd service | `bitbucket.service` |
| Deployment setting | Data Center / clustered |
| SSH validation | Disabled during initial setup |

The successful validation run reported:

```text
Bitbucket validation completed
Version           : 10.4.2
Install Directory : /app/bitbucket-10.4.2
Home Directory    : /app/bitbucket-data
HTTP Port         : 7990
SSH Port          : 7999
Database          : bitbucket
PostgreSQL        : 127.0.0.1:15432
Service           : bitbucket
Service State     : active
Service Enabled   : enabled
Clustered         : True
```

The application was reachable on port `7990`. Before completion of the Bitbucket web setup, an HTTP redirect such as the following is expected and demonstrates that the web application is responding:

```text
HTTP/1.1 302
Location: http://localhost:7990/unavailable?next=%2Fmvc%2Fhome
```

## Directory Structure

```text
bitbucket-dc-ansible/
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   └── bitbucket.yml
│   └── hosts.yml
├── playbooks
│   ├── install_bitbucket.yml
│   ├── uninstall_bitbucket.yml
│   └── validate_bitbucket.yml
└── roles
    └── bitbucket
        ├── defaults
        │   └── main.yml
        ├── files
        │   ├── atlassian-bitbucket-10.4.2-x64.bin
        │   └── atlassian-bitbucket-10.4.2.tar.gz
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── tasks
        │   ├── cluster.yml
        │   ├── configure.yml
        │   ├── database.yml
        │   ├── install.yml
        │   ├── install_archive.yml
        │   ├── install_installer.yml
        │   ├── main.yml
        │   ├── precheck.yml
        │   ├── prerequisites.yml
        │   ├── systemd.yml
        │   ├── uninstall.yml
        │   └── validate.yml
        └── templates
            ├── bitbucket.properties.j2
            ├── bitbucket.service.j2
            └── response.varfile.j2
```

Before committing, also remove any accidental repository-root files such as `-` or `0` if they are not intentionally part of the project.

## Requirements

### Control node

- Ansible 2.15 or later recommended.
- Python 3.
- Bitbucket installation media under `roles/bitbucket/files/`.
- Privilege escalation capability when managing the local or remote target.

### Managed node

The tested managed-node platform is:

```text
Red Hat Enterprise Linux 9.8
x86_64
```

The host requires:

- sufficient free space under `/app`;
- root or `become` access;
- PostgreSQL network connectivity;
- a supported Git installation;
- required operating-system packages;
- access to the configured shared filesystem when implementing a multi-node Data Center deployment.

The tested precheck reported:

```text
OS               : RedHat 9.8
Architecture     : x86_64
Free /app Space  : 13.65 GB
Install Method   : installer
Install Media    : atlassian-bitbucket-10.4.2-x64.bin
Database         : 127.0.0.1:15432/bitbucket
HTTP Port        : 7990
SSH Port         : 7999
Clustered        : True
```

### Database

The PostgreSQL database and role must exist before application deployment.

Tested values:

```yaml
postgres_host: "127.0.0.1"
postgres_port: 15432
postgres_database: bitbucket
postgres_user: bitbucket
postgres_password: "bitbucket"
```

The current database lifecycle is intentionally external to this repository. The PostgreSQL database is **not removed** by the Bitbucket uninstall role.

For production, do not store the password as plaintext. Use Ansible Vault or another approved secrets-management mechanism.

## Installation Methods

### Linux Binary Installer Method

Recommended for the currently tested deployment:

```yaml
bitbucket_install_method: "installer"
```

Installation media:

```text
roles/bitbucket/files/atlassian-bitbucket-10.4.2-x64.bin
```

The installer was manually exercised to establish the correct Install4j response-file settings.

The tested interactive choices were:

```text
Install a new instance
Bitbucket Data Center instance
Installation Directory: /var/tmp/bitbucket-bin-test
Home Directory: /var/tmp/bitbucket-bin-test-home
HTTP Port: 7990
Install as service: No
Launch Bitbucket: No
```

The automation reproduces these choices using `response.varfile.j2`, but substitutes the Ansible-managed production paths.

### Archive Method

Set:

```yaml
bitbucket_install_method: "archive"
```

Installation media:

```text
roles/bitbucket/files/atlassian-bitbucket-10.4.2.tar.gz
```

The archive-specific workflow is implemented in:

```text
roles/bitbucket/tasks/install_archive.yml
```

Both installation methods ultimately converge on:

```text
database.yml
configure.yml
cluster.yml
systemd.yml
validate.yml
```

This means the installation mechanism does not determine the long-term service-management model.

## Preparing the Installers

Place both installation files under:

```text
roles/bitbucket/files/
```

Expected files:

```text
atlassian-bitbucket-10.4.2-x64.bin
atlassian-bitbucket-10.4.2.tar.gz
```

Verify them:

```bash
ls -lh roles/bitbucket/files/

file \
  roles/bitbucket/files/atlassian-bitbucket-10.4.2-x64.bin

sha256sum \
  roles/bitbucket/files/atlassian-bitbucket-10.4.2-x64.bin
```

The tested Linux installer checksum is:

```text
4b3f5026cf3afd1a60d0153840dc377433e8a3a6b358bfb258e43e9c3b2c0d27
```

The installer should identify as:

```text
POSIX shell script executable (binary data)
```

The Ansible precheck should validate only the media required for the selected installation method.

## Inventory

Current local inventory:

```yaml
---
all:
  children:
    bitbucket:
      hosts:
        localhost:
          ansible_connection: local
```

For remote deployment, an inventory can instead resemble:

```yaml
all:
  children:
    bitbucket:
      hosts:
        bitbucket01.example.com:
          ansible_host: 10.1.20.40
          ansible_user: ansible
```

For multiple Data Center nodes, add additional hosts and use host-specific variables where node-specific configuration is required.

## Configuration Variables

The primary environment configuration is:

```text
inventory/group_vars/bitbucket.yml
```

### Product

```yaml
bitbucket_version: "10.4.2"

bitbucket_user: bitbucket
bitbucket_group: bitbucket
```

### Installation method

A dual-method implementation should define:

```yaml
bitbucket_install_method: "installer"
```

Supported values:

```text
installer
archive
```

### Paths

```yaml
bitbucket_base_dir: /app

bitbucket_install_dir: "/app/bitbucket-{{ bitbucket_version }}"
bitbucket_home: /app/bitbucket-data

bitbucket_shared_home_enabled: true
bitbucket_shared_home: "{{ bitbucket_home }}/shared"
```

The tested effective paths are:

```text
/app/bitbucket-10.4.2
/app/bitbucket-data
/app/bitbucket-data/shared
```

### Installation media

Example binary variables:

```yaml
bitbucket_installer: "atlassian-bitbucket-{{ bitbucket_version }}-x64.bin"

bitbucket_installer_sha256: "4b3f5026cf3afd1a60d0153840dc377433e8a3a6b358bfb258e43e9c3b2c0d27"
```

Archive:

```yaml
bitbucket_archive: "atlassian-bitbucket-{{ bitbucket_version }}.tar.gz"
```

Keep the archive SHA256 in inventory/defaults when archive validation is enabled.

### Java

The original environment defined:

```yaml
bitbucket_java_package: java-21-openjdk

bitbucket_java_home: /usr/lib/jvm/java-21-openjdk-21.0.12.0.8-1.2.el9.x86_64
```

However, the tested Bitbucket binary installation started using the installer-bundled runtime:

```text
/app/bitbucket-10.4.2/jre/bin/java
```

Do not assume the configured operating-system Java path is the actual Bitbucket runtime when the `.bin` installer includes its own JRE. Verify the effective process after installation.

### Network

```yaml
bitbucket_http_port: 7990
bitbucket_ssh_port: 7999
```

The HTTP port was observed listening successfully.

SSH validation is initially disabled:

```yaml
bitbucket_ssh_validation_enabled: false
```

This is appropriate before the Bitbucket setup wizard and SSH service configuration are complete.

### PostgreSQL

```yaml
postgres_version: 17

postgres_host: "127.0.0.1"
postgres_port: 15432

postgres_database: bitbucket
postgres_user: bitbucket
postgres_password: "bitbucket"
```

### JVM

```yaml
bitbucket_jvm_minimum_memory: 2g
bitbucket_jvm_maximum_memory: 4g
```

The observed process contained:

```text
-Xms2g
-Xmx4g
-XX:+UseG1GC
```

### Data Center

```yaml
bitbucket_clustered: true
bitbucket_shared_home_enabled: true
bitbucket_shared_home: "{{ bitbucket_home }}/shared"
```

The current test uses:

```text
/app/bitbucket-data/shared
```

This is valid for the current single-node functional test but should not be interpreted as production multi-node shared storage.

### Service

```yaml
bitbucket_service_name: bitbucket
```

The deployed unit is:

```text
/etc/systemd/system/bitbucket.service
```

### Validation

```yaml
bitbucket_validation_url: "http://127.0.0.1:7990"
bitbucket_validation_retries: 60
bitbucket_validation_delay: 10

bitbucket_ssh_validation_enabled: false
```

### Uninstall

Current safe default:

```yaml
bitbucket_purge_data: false
```

Meaning:

```text
Application binaries    REMOVED
systemd service         REMOVED
BITBUCKET_HOME          PRESERVED
Shared data             PRESERVED
OS user                 PRESERVED
OS group                PRESERVED
PostgreSQL database     PRESERVED
PostgreSQL role         PRESERVED
```

Use data purge only when deliberate destruction of application data is intended.

## Usage

Run commands from the repository root.

### Syntax Check

Installation:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_bitbucket.yml \
  --syntax-check
```

Uninstall:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/uninstall_bitbucket.yml \
  --syntax-check
```

Validation:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/validate_bitbucket.yml \
  --syntax-check
```

### List Tasks

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_bitbucket.yml \
  --list-tasks
```

### Precheck

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_bitbucket.yml \
  --tags precheck
```

The tested precheck validates the deployment environment before installation.

### Full Installation

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_bitbucket.yml
```

Expected workflow:

```text
precheck
prerequisites
install
database
configure
cluster
systemd
validate
```

### Validate Existing Installation

A dedicated validation playbook exists:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/validate_bitbucket.yml
```

This is useful after:

- application startup;
- server restart;
- configuration changes;
- database maintenance;
- network changes;
- manual troubleshooting.

### Uninstall

Safe/default uninstall:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/uninstall_bitbucket.yml
```

This preserves Bitbucket application data and the PostgreSQL database.

For an intentional purge, use the supported purge variable only after reviewing the target paths:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/uninstall_bitbucket.yml \
  -e bitbucket_purge_data=true
```

**Warning:** purge mode can remove `BITBUCKET_HOME` and the Bitbucket OS account. Treat it as destructive.

## Role Task Flow

The role is structured around the following task files.

| Task file | Purpose |
|---|---|
| `precheck.yml` | Validate OS, architecture, disk, installer, database, ports and prerequisites |
| `prerequisites.yml` | Install prerequisites and establish Bitbucket account/directories |
| `install.yml` | Dispatch selected installation method |
| `install_archive.yml` | Install from Atlassian `.tar.gz` |
| `install_installer.yml` | Install from Atlassian Linux `.bin` |
| `database.yml` | Validate Bitbucket database requirements/connectivity |
| `configure.yml` | Configure BITBUCKET_HOME and `bitbucket.properties` |
| `cluster.yml` | Validate/configure Data Center shared-home behavior |
| `systemd.yml` | Deploy and manage `bitbucket.service` |
| `validate.yml` | Perform post-install runtime validation |
| `uninstall.yml` | Remove application safely while preserving data by default |

The conceptual execution path is:

```text
roles/bitbucket/tasks/main.yml
            |
            +--> precheck.yml
            |
            +--> prerequisites.yml
            |
            +--> install.yml
            |       |
            |       +--> install_installer.yml
            |       |
            |       +--> install_archive.yml
            |
            +--> database.yml
            |
            +--> configure.yml
            |
            +--> cluster.yml
            |
            +--> systemd.yml
            |
            +--> validate.yml
```

## Binary Installer Response File

A successful manual Bitbucket 10.4.2 installation generated:

```text
/var/tmp/bitbucket-bin-test/.install4j/response.varfile
```

with:

```text
# install4j response file for Bitbucket 10.4.2
app.bitbucketHome=/var/tmp/bitbucket-bin-test-home
app.defaultInstallDir=/var/tmp/bitbucket-bin-test
app.install.service$Boolean=false
httpPort=7990
installation.type=INSTALL
launch.application$Boolean=false
sys.adminRights$Boolean=true
sys.adminRightsUiRootUnix$Boolean=false
sys.languageId=en
```

The Ansible template is:

```text
roles/bitbucket/templates/response.varfile.j2
```

The production response file should substitute Ansible variables for the test paths, conceptually:

```text
app.bitbucketHome={{ bitbucket_home }}
app.defaultInstallDir={{ bitbucket_install_dir }}
app.install.service$Boolean=false
httpPort={{ bitbucket_http_port }}
installation.type=INSTALL
launch.application$Boolean=false
sys.adminRights$Boolean=true
sys.adminRightsUiRootUnix$Boolean=false
sys.languageId=en
```

Two settings are especially important:

```text
app.install.service$Boolean=false
launch.application$Boolean=false
```

They prevent the Atlassian installer from taking ownership of service creation or application startup.

The lifecycle remains:

```text
Atlassian .bin installer
        |
        v
Install Bitbucket files
        |
        v
Ansible configuration
        |
        v
Ansible systemd unit
        |
        v
Start Bitbucket
        |
        v
Validate
```

## Database Configuration

The Ansible-managed template is:

```text
roles/bitbucket/templates/bitbucket.properties.j2
```

The tested database portion is:

```properties
jdbc.driver=org.postgresql.Driver
jdbc.url=jdbc:postgresql://127.0.0.1:15432/bitbucket
jdbc.user=bitbucket
jdbc.password=bitbucket
```

The tested running application established PostgreSQL sessions:

```text
datname   : bitbucket
usename   : bitbucket
client    : 127.0.0.1
state     : idle
```

Multiple idle connections were observed, confirming that the Bitbucket process was connecting to the configured PostgreSQL database.

The database is considered an external dependency. Its creation, backup, restore, upgrade and removal should be handled independently, for example through the companion PostgreSQL automation.

## Bitbucket Home and Shared Home

The tested Bitbucket home is:

```text
/app/bitbucket-data
```

The active shared content is located under:

```text
/app/bitbucket-data/shared
```

Observed content included:

```text
secured/
keys/
config/
data/
plugins/
bitbucket.properties
secrets-config.yaml
.lock
```

This is important because Bitbucket creates and manages sensitive runtime content in the shared area, including encryption/signing material and repository-related data.

The current Ansible configuration therefore uses:

```yaml
bitbucket_shared_home: "{{ bitbucket_home }}/shared"
```

and not a second unrelated local path such as:

```text
/app/bitbucket-shared
```

For a real multi-node Data Center deployment, the shared location must be designed as shared storage accessible by all participating nodes according to the production architecture.

Do not simply create the same local path independently on each server and treat it as shared storage.

## Systemd Ownership Model

The binary installer is instructed not to install Bitbucket as a service:

```text
app.install.service$Boolean=false
```

Ansible deploys:

```text
roles/bitbucket/templates/bitbucket.service.j2
```

to:

```text
/etc/systemd/system/bitbucket.service
```

The tested service state was:

```text
Loaded: loaded
Active: active (running)
Enabled: enabled
```

The process ran under the dedicated Bitbucket account and used:

```text
-Dbitbucket.home=/app/bitbucket-data
-Dbitbucket.install=/app/bitbucket-10.4.2
-Xms2g
-Xmx4g
-XX:+UseG1GC
```

This architecture provides one lifecycle regardless of installation method:

```text
.tar.gz -> Ansible systemd
.bin    -> Ansible systemd
```

## Data Center / Cluster Mode

The current configuration enables:

```yaml
bitbucket_clustered: true
bitbucket_shared_home_enabled: true
bitbucket_shared_home: "{{ bitbucket_home }}/shared"
```

The cluster task validates the shared-home path.

For the current single-node test, the local shared directory is sufficient to exercise the role structure.

For production multi-node Data Center, additional architecture is required. In particular:

- shared storage must actually be shared across nodes;
- all nodes must use the same database;
- node-local and shared data responsibilities must be clearly separated;
- load balancing must be configured;
- network connectivity between cluster members must be allowed;
- Bitbucket's shared search requirements must be addressed;
- backup and restore procedures must include both database and filesystem data.

The installer itself explicitly reported that a complete Data Center installation requires:

```text
Shared database system
Shared search server
```

and, for clustered deployment:

```text
shared file system
```

Therefore `bitbucket_clustered: true` should be understood as enabling the Ansible Data Center configuration path; it does not by itself turn a local single-node test into a production-ready cluster.

## Validation

The automation has a dedicated validation playbook:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/validate_bitbucket.yml
```

The tested full installation completed successfully:

```text
PLAY RECAP
localhost : ok=110 changed=16 unreachable=0 failed=0 skipped=7 rescued=0 ignored=0
```

### Service Validation

```bash
systemctl status bitbucket --no-pager -l
```

Expected:

```text
Active: active (running)
```

### Process Validation

```bash
ps -ef | grep '[b]itbucket'
```

The tested Java launcher was:

```text
com.atlassian.bitbucket.internal.launcher.BitbucketServerLauncher start
```

### Port Validation

```bash
ss -lntp | grep -E ':7990|:7999'
```

During the initial setup test, HTTP port `7990` was listening.

Port `7999` may not listen until Bitbucket SSH is fully initialized/configured. This is why:

```yaml
bitbucket_ssh_validation_enabled: false
```

is appropriate during initial deployment.

### HTTP Validation

```bash
curl -sS -I --max-time 10 \
  http://localhost:7990/ | head -15
```

A `200` or setup-related `302` response demonstrates HTTP availability.

### Database Validation

Example:

```bash
sudo -u postgres psql -p 15432 -d postgres -c \
"SELECT datname, usename, client_addr, state
 FROM pg_stat_activity
 WHERE datname='bitbucket';"
```

### Filesystem Validation

```bash
stat -c '%U:%G %a %n' \
  /app/bitbucket-10.4.2 \
  /app/bitbucket-data \
  /app/bitbucket-data/shared
```

The tested ownership was:

```text
bitbucket:bitbucket
```

## Uninstall Behavior

The uninstall workflow is intentionally conservative by default.

Configuration:

```yaml
bitbucket_purge_data: false
```

The tested uninstall produced:

```text
Bitbucket uninstall completed
Install Dir : REMOVED
Home Dir    : PRESERVED
Shared Home : PRESERVED
Service     : REMOVED
OS User     : PRESERVED
OS Group    : PRESERVED
Database    : NOT MODIFIED
DB User     : NOT MODIFIED
```

The post-uninstall state was independently verified:

```text
bitbucket.service             -> absent
/app/bitbucket-10.4.2         -> absent
/app/bitbucket-data           -> present
/app/bitbucket-data/shared    -> present
bitbucket OS user             -> present
bitbucket OS group            -> present
Bitbucket process             -> none
port 7990                     -> not listening
port 7999                     -> not listening
PostgreSQL bitbucket DB       -> present
PostgreSQL bitbucket role     -> present
```

This is the preferred behavior for normal application reinstallation or application-binary replacement because valuable Bitbucket data remains intact.

### Purge Mode

When explicitly requested:

```yaml
bitbucket_purge_data: true
```

the uninstall role may additionally remove:

```text
BITBUCKET_HOME
Bitbucket service user
Bitbucket service group
```

The PostgreSQL database remains outside this role's lifecycle.

Use purge mode only when filesystem data destruction is intentional and backups have been verified.

## Reinstallation

Because the default uninstall preserves:

```text
/app/bitbucket-data
```

a later installation can reuse the existing application data.

Typical reinstall flow:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_bitbucket.yml
```

The installation task should recreate:

```text
/app/bitbucket-10.4.2
/etc/systemd/system/bitbucket.service
```

while retaining the existing Bitbucket home and database.

Before performing this workflow in production, ensure the Bitbucket version is compatible with the existing home/database schema. Do not use application reinstallation as an uncontrolled downgrade mechanism.

## Troubleshooting

### Precheck runs only Gathering Facts

If:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_bitbucket.yml \
  --tags precheck
```

shows only `Gathering Facts`, ensure the relevant tasks/includes in `roles/bitbucket/tasks/main.yml` and `precheck.yml` carry the expected tag behavior.

After correcting the task tagging, the tested precheck ran successfully and reported 24 successful tasks.

### Installer not found

Check:

```bash
ls -lh roles/bitbucket/files/
```

Expected:

```text
atlassian-bitbucket-10.4.2-x64.bin
atlassian-bitbucket-10.4.2.tar.gz
```

Ensure the selected installation method points to the correct filename.

### Installer checksum failure

Run:

```bash
sha256sum \
  roles/bitbucket/files/atlassian-bitbucket-10.4.2-x64.bin
```

Expected for the tested installer:

```text
4b3f5026cf3afd1a60d0153840dc377433e8a3a6b358bfb258e43e9c3b2c0d27
```

### Bitbucket service does not start

Check:

```bash
systemctl status bitbucket --no-pager -l
journalctl -u bitbucket -n 100 --no-pager
```

Also inspect:

```text
/app/bitbucket-data/log/atlassian-bitbucket.log
```

### Port 7990 not listening

Check:

```bash
ss -lntp | grep ':7990'

ps -ef | grep '[b]itbucket'

tail -100 \
  /app/bitbucket-data/log/atlassian-bitbucket.log
```

### HTTP returns 302

During setup, a redirect is not necessarily a failure.

The tested initial deployment returned:

```text
HTTP/1.1 302
Location: http://localhost:7990/unavailable?next=%2Fmvc%2Fhome
```

Interpret HTTP validation together with service state, process state, logs and setup state.

### SSH port 7999 is not listening

The initial installation may not expose Bitbucket SSH immediately.

Keep:

```yaml
bitbucket_ssh_validation_enabled: false
```

until the web setup and SSH configuration are complete.

After enabling/configuring SSH, set it to `true` and validate port `7999`.

### Shared home path mismatch

Use:

```bash
grep -RniE \
  'bitbucket_home.*/shared|bitbucket_shared_home|bitbucket-shared' \
  roles/bitbucket inventory
```

The current configuration should consistently resolve to:

```text
/app/bitbucket-data/shared
```

unless you intentionally configure an external shared filesystem.

### Database connection issues

Check reachability:

```bash
nc -zv 127.0.0.1 15432
```

Check database:

```bash
sudo -u postgres psql -p 15432 -lqt | grep bitbucket
```

Check active connections:

```bash
sudo -u postgres psql -p 15432 -d postgres -c \
"SELECT datname, usename, client_addr, state
 FROM pg_stat_activity
 WHERE datname='bitbucket';"
```

### Unexpected old Bitbucket home

The tested environment already contained `/app/bitbucket-data` from an earlier installation even when the application binaries and service were absent.

Before a clean-room test, decide explicitly whether you want:

```text
preserve existing home
```

or:

```text
purge existing home
```

Do not delete an existing Bitbucket home merely because `/app/bitbucket-10.4.2` is absent.

## Security Recommendations

### Protect Database Credentials

Do not retain:

```yaml
postgres_password: "bitbucket"
```

in production plaintext inventory.

Use Ansible Vault, for example:

```bash
ansible-vault encrypt_string \
  'REPLACE_WITH_REAL_PASSWORD' \
  --name 'postgres_password'
```

### Protect Bitbucket Shared Data

The directory:

```text
/app/bitbucket-data/shared
```

contains sensitive application data, keys, secrets and repository-related content.

Restrict access to the Bitbucket service account and approved administrators.

### Do Not Commit Installation Media

Atlassian binary distributions are large and should normally not be committed directly to Git.

Example `.gitignore`:

```gitignore
roles/bitbucket/files/*.bin
roles/bitbucket/files/*.tar.gz
```

Use an approved artifact repository or controlled software-distribution mechanism for production automation.

### Protect Secrets Generated by Bitbucket

Observed runtime files include:

```text
/app/bitbucket-data/shared/secrets-config.yaml
/app/bitbucket-data/shared/keys/
/app/bitbucket-data/shared/config/secret-system-signing-key.asc
```

These should be included in appropriate backup/security procedures and must not be exposed through source control.

### Restrict Network Ports

Expose only required ports.

Typical application ports in this automation:

```text
7990/tcp - Bitbucket HTTP
7999/tcp - Bitbucket SSH
15432/tcp - PostgreSQL in the tested environment
```

Database access should be restricted to approved Bitbucket nodes and administrative systems.

## Tested Lifecycle

The following lifecycle has been exercised.

### Installer Investigation

```text
PASS - Bitbucket 10.4.2 Linux installer executable
PASS - interactive installation
PASS - response.varfile discovery
PASS - unattended-response parameters identified
PASS - custom install directory
PASS - custom home directory
PASS - HTTP 7990
PASS - installer service creation disabled
PASS - automatic startup disabled
```

### Ansible Precheck

```text
PASS - RHEL 9.8
PASS - x86_64
PASS - /app free-space check
PASS - installer method selection
PASS - binary media validation
PASS - PostgreSQL endpoint validation
PASS - HTTP/SSH port configuration validation
PASS - Data Center configuration validation
```

### Installation

```text
PASS - binary installer deployment
PASS - /app/bitbucket-10.4.2 creation
PASS - /app/bitbucket-data use
PASS - bitbucket user/group ownership
PASS - bitbucket.properties deployment
PASS - PostgreSQL connectivity
PASS - systemd service deployment
PASS - service enablement
PASS - service startup
PASS - Java process
PASS - HTTP port 7990
PASS - HTTP response
PASS - validation summary
```

The tested full installation recap was:

```text
localhost : ok=110 changed=16 unreachable=0 failed=0 skipped=7 rescued=0 ignored=0
```

### Runtime

Observed:

```text
bitbucket.service : active (running)
HTTP 7990         : LISTEN
PostgreSQL        : connected
Install owner     : bitbucket:bitbucket
Home owner        : bitbucket:bitbucket
```

### Uninstall

```text
PASS - service stopped
PASS - service disabled
PASS - service unit removed
PASS - installation directory removed
PASS - process stopped
PASS - HTTP/SSH ports closed
PASS - BITBUCKET_HOME preserved
PASS - shared data preserved
PASS - bitbucket user preserved
PASS - bitbucket group preserved
PASS - PostgreSQL database preserved
PASS - PostgreSQL role preserved
```

The tested uninstall recap was:

```text
localhost : ok=16 changed=4 unreachable=0 failed=0 skipped=4 rescued=0 ignored=0
```

## Production Readiness Notes

The current automation provides a strong application lifecycle foundation, but several items should be completed before treating the deployment as production-ready.

### Complete Web Setup

After installation:

```text
http://<bitbucket-host>:7990
```

Complete the Bitbucket setup wizard and apply the appropriate Data Center license.

### Shared Filesystem

The current test uses:

```text
/app/bitbucket-data/shared
```

on one host.

For multiple Bitbucket Data Center nodes, implement an actual shared filesystem and update:

```yaml
bitbucket_shared_home
```

accordingly.

### Shared Search

The Bitbucket 10.4.2 installer explicitly indicates that a complete Data Center deployment requires a shared search server. Design and deploy the supported search architecture before production cluster rollout.

### Reverse Proxy / Load Balancer

Production access should normally be fronted by the organization's approved reverse proxy or load balancer, with TLS and appropriate forwarding configuration.

### Backup

A Bitbucket recovery strategy must account for both:

```text
PostgreSQL database
Bitbucket filesystem/shared data
```

Backups must be coordinated so that application and database state can be restored consistently.

### Monitoring

At minimum monitor:

```text
bitbucket.service
HTTP availability
JVM memory
filesystem capacity
PostgreSQL connectivity
application errors
repository/storage capacity
shared filesystem availability
search service health
```

### Secrets

Move database credentials and any other sensitive values out of plaintext inventory before production deployment.

## Summary

This repository provides repeatable Ansible automation for the Bitbucket Data Center application lifecycle on RHEL 9.

The tested reference deployment is:

```text
Bitbucket Data Center 10.4.2
RHEL 9.8
Linux x64 binary installer
/app/bitbucket-10.4.2
BITBUCKET_HOME=/app/bitbucket-data
Shared=/app/bitbucket-data/shared
HTTP=7990
SSH=7999
PostgreSQL=127.0.0.1:15432/bitbucket
JVM heap=2g/4g
systemd=bitbucket.service
Data Center mode enabled
```

The recommended installation method for the currently validated workflow is:

```yaml
bitbucket_install_method: "installer"
```

The safe default uninstall behavior is:

```yaml
bitbucket_purge_data: false
```

which removes the application binaries and systemd service while preserving Bitbucket home, shared data, the OS account and PostgreSQL data.

This makes the automation suitable for controlled installation, validation, removal and reinstallation workflows while keeping persistent application data outside the normal application-binary lifecycle.

For a production multi-node Data Center deployment, complete the Bitbucket web setup and licensing, replace local shared storage with a real shared filesystem, deploy the required shared search architecture, secure credentials, configure TLS/load balancing, and implement coordinated database/filesystem backup and monitoring.
