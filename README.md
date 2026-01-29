# PrivX Upgrade Automation – Ansible

[![Ansible Tests](https://img.shields.io/github/actions/workflow/status/SSHcom/ansible-upgrade-privx/ansible-tests.yml?style=for-the-badge&label=Ansible%20Tests)](https://github.com/SSHcom/ansible-upgrade-privx/actions)
[![Last Commit](https://img.shields.io/github/last-commit/SSHcom/ansible-upgrade-privx.svg?style=for-the-badge)](https://github.com/SSHcom/ansible-upgrade-privx/commits)

This repository contains Ansible-based upgrade processes for multi-node PrivX deployments. Two upgrade methods are available:

- **Staged Upgrade** (`upgrade_privx.yml`) - Traditional upgrade with manual validation gates and explicit operator control at each stage
- **Zero Downtime Upgrade (ZDU)** (`upgrade_privx_zdu.yml`) - Automated upgrade using PrivX's built-in ZDU scripts with no service interruption

Both processes are designed for safe production upgrades with proper validation and rollback readiness.

---

## Overview

The upgrade process is now split into two separate playbooks for better organization:

### PrivX Core Upgrade (`upgrade_privx.yml`):
1. **Backup validation** (mandatory confirmation)
2. **RPM availability validation** (check target version exists on primary node)
3. **Stop PrivX on all secondary nodes** 
4. **Upgrade PrivX on primary node**
5. **Manual validation of primary node**
6. **Upgrade PrivX on secondary nodes**

### PrivX Zero Downtime Upgrade (`upgrade_privx_zdu.yml`) - Alternative:
1. **Backup validation** (mandatory confirmation)
2. **ZDU version compatibility check** (max one major version behind)
3. **RPM availability validation** (check target version exists on all nodes)
4. **Execute upgrade_first_stage.sh on all nodes** (with 30s stabilization wait)
5. **Validate all nodes have target version**
6. **Execute upgrade_second_stage.sh on primary node only**

### PrivX Extender Upgrade (`upgrade_extenders.yml`) - Optional:
1. **Extender configuration file validation** (check required config files exist)
2. **Extender RPM availability validation** (check on first extender node)
3. **Configuration backup and upgrade process** (backup → upgrade → deploy config → postinstall)

### PrivX Web Access Gateway Upgrade (`upgrade_wag.yml`) - Optional:
1. **WAG configuration file validation** (check carrier and webproxy config files exist)
2. **WAG RPM availability validation** (check PrivX-Carrier and PrivX-Web-Proxy packages)
3. **Paired upgrade process** (upgrade carrier → webproxy pairs sequentially)

**⚠️ CRITICAL: WAG Pairing Requirements**
- Carrier and Web-Proxy nodes work in pairs and must be configured correctly
- Nodes must be listed in the same order in both `[privxcarrier]` and `[privxwebproxy]` groups
- Incorrect pairing can result in service downtime during upgrades

---

## Assumptions

- **PrivX repository is enabled on all machines**
- **PrivX-Extender repository is enabled on extender machines** (if using extenders)
- **PrivX-Carrier and PrivX-Web-Proxy repositories are enabled on WAG machines** (if using WAG)
- PrivX is assumed to automatically start after the package upgrade
- PrivX-Extender works in pairs for High Availability and must be upgraded one by one
- **PrivX-Carrier and PrivX-Web-Proxy work in pairs and must be configured in matching order**
- The playbooks do not explicitly start the services

---

## Inventory Structure

Inventory must define separate host groups:

```ini
[privx_primary]
privx-node1.example.com

[privx_additional_nodes]
privx-node2.example.com
privx-node3.example.com
privx-node4.example.com

[privxextender]
# Add your PrivX Extender nodes here
# extender-node1.example.com
# extender-node2.example.com

[privxcarrier]
# Add your PrivX Carrier nodes here
# ⚠️ CRITICAL: List nodes in pairing sequence!
# carrier-node1.example.com
# carrier-node2.example.com

[privxwebproxy]
# Add your PrivX WebProxy nodes here
# ⚠️ CRITICAL: Must match Carrier node order!
# webproxy-node1.example.com  # Pairs with carrier-node1
# webproxy-node2.example.com  # Pairs with carrier-node2

[all:vars]
privx_target_version=40.0
privx_validation_file="/tmp/privx_validation_done-{{ privx_target_version }}"
extender_stabilization_wait=30
wag_pair_stabilization_wait=30
zdu_stabilization_wait=30
REMOTE_USER=rocky
BECOME=yes
ansible_ssh_extra_args='-o StrictHostKeyChecking=no'
```

---

## Repository Structure

```text
.
├── upgrade_privx.yml          # Main PrivX upgrade playbook
├── upgrade_privx_zdu.yml      # PrivX Zero Downtime Upgrade playbook
├── upgrade_extenders.yml      # PrivX-Extender upgrade playbook
├── upgrade_wag.yml            # PrivX Web Access Gateway upgrade playbook
├── inventory/
│   └── hosts.ini             # Inventory file with host groups
├── configuration_files/       # Component configuration files
│   └── README.md             # Configuration file instructions
└── tasks/                    # Individual task files (organized by component)
    ├── upgrade_privx/        # PrivX core upgrade tasks
    │   ├── backup_validation.yml
    │   ├── rpm_validation.yml
    │   ├── stop_secondary.yml
    │   ├── upgrade_primary.yml
    │   ├── upgrade_secondary.yml
    │   ├── version_check.yml
    │   ├── zdu_version_check.yml
    │   ├── zdu_first_stage.yml
    │   ├── zdu_version_validation.yml
    │   └── zdu_second_stage.yml
    ├── upgrade_extender/     # PrivX-Extender upgrade tasks
    │   ├── extender_config_validation.yml
    │   ├── extender_rpm_validation.yml
    │   └── upgrade_extenders.yml
    └── upgrade_wag/          # PrivX Web Access Gateway upgrade tasks
        ├── wag_config_validation.yml
        ├── wag_rpm_validation.yml
        ├── upgrade_wag_pair.yml
        ├── upgrade_carrier.yml
        └── upgrade_webproxy.yml
```

---

## Configuration Variables

### Required

- `privx_target_version`  
  Target PrivX version to upgrade to (defined in inventory).
  **⚠️ IMPORTANT**: Consult the "Supported upgrade paths" section in the PrivX product release notes before setting this value. Not all version upgrades are supported directly - some may require intermediate upgrades.

- `privx_validation_file`  
  Path to validation marker file (defined in inventory).

### Safety / Control Variables

- `privx_backup_confirmed` (boolean, default: false)  
  Must be explicitly set to `true` to proceed with the primary upgrade.


### ZDU Configuration Settings

- `zdu_stabilization_wait` (integer, default: 30)  
  Wait time in seconds for system stabilization after first stage upgrade execution.  
  Applied once globally after all nodes complete first stage.

### ZDU Version Compatibility

- **Maximum version gap**: One major version behind target
- **Valid upgrade paths**: 
  - `40.x → 41.x` (major version upgrade)
  - `41.0 → 41.5` (minor version upgrade)
  - `41.2 → 41.8` (patch version upgrade)
- **Invalid upgrade paths**:
  - `39.x → 41.x` (more than one major version gap)
  - Use standard upgrade process for incompatible version gaps

### Extender Configuration Files

For extender upgrades, configuration files are required in the `configuration_files/` directory:
- **Naming convention**: `<inventory_hostname>-extender-config.toml`
- **Source**: Downloaded from PrivX UI after core upgrade
- **Timing**: Must be obtained after PrivX core upgrade completion
- **Manual changes**: Apply any necessary configuration modifications before saving

**ℹ️ Extender Configuration Versions**

PrivX supports two extender configuration formats, both fully supported in PrivX 42.0 and future versions:

**V1 Configuration:**
```toml
extender_mode = ""
```
- Traditional extender configuration format
- Fully supported in all PrivX versions including 42.0+
- Suitable for standard extender deployments

**V2 Configuration (Available in PrivX 42.0+):**
```toml
extender_mode = "normal"   # Standard extender mode
extender_mode = "forward"  # Forward mode
extender_mode = "passive"  # Passive mode
```
- Introduces new operating modes for advanced use cases
- Provides additional flexibility for complex deployments
- Optional - use only if your deployment requires these modes

**Configuration Selection:**
- **Both V1 and V2 are fully supported** - choose based on your use case
- Download the appropriate configuration from PrivX UI after core upgrade
- The playbook will detect and display which version you're using
- No action required if using V1 - it's fully compatible with PrivX 42.0+

**Example**: If your inventory has:
```ini
[privxextender]
extender-node1.example.com
extender-node2.example.com
```

You need these files:
```
configuration_files/extender-node1.example.com-extender-config.toml
configuration_files/extender-node2.example.com-extender-config.toml
```

### Extender Upgrade Settings

- `extender_stabilization_wait` (integer, default: 30)  
  Wait time in seconds for extender service to stabilize after upgrade and configuration deployment.  
  **Note**: Only applies when there are multiple extender nodes (for High Availability scenarios).

### Extender Postinstall Timeout

- **Timeout**: 120 seconds (2 minutes)
- **Behavior**: If the extender postinstall script times out or fails:
  - Script output (stdout/stderr) is displayed
  - Detailed troubleshooting instructions are provided
  - Playbook execution stops for manual intervention
  - User must resolve the issue before continuing

### Extender Backup Behavior

- **Version-specific backups**: Each upgrade version creates its own backup file
- **Naming convention**: `extender-config.toml.backup-pre-upgrade-<version>`
- **Idempotent**: Only creates backup once per version (safe to re-run)
- **Example**: `extender-config.toml.backup-pre-upgrade-40.0`

### WAG Configuration Files

For WAG upgrades, configuration files are required in the `configuration_files/` directory:
- **Carrier naming convention**: `<inventory_hostname>-carrier-config.toml`
- **WebProxy naming convention**: `<inventory_hostname>-web-proxy-config.toml`
- **Source**: Downloaded from PrivX UI after core upgrade
- **Timing**: Must be obtained after PrivX core upgrade completion
- **Manual changes**: Apply any necessary configuration modifications before saving

**Example**: If your inventory has:
```ini
[privxcarrier]
carrier-site1.example.com
carrier-site2.example.com

[privxwebproxy]
webproxy-site1.example.com
webproxy-site2.example.com
```

You need these files:
```
configuration_files/carrier-site1.example.com-carrier-config.toml
configuration_files/carrier-site2.example.com-carrier-config.toml
configuration_files/webproxy-site1.example.com-web-proxy-config.toml
configuration_files/webproxy-site2.example.com-web-proxy-config.toml
```

### WAG Backup Behavior

- **Version-specific backups**: Each upgrade version creates its own backup files
- **Carrier naming**: `carrier-config.toml.backup-pre-upgrade-<version>`
- **WebProxy naming**: `web-proxy-config.toml.backup-pre-upgrade-<version>`
- **Idempotent**: Only creates backup once per version (safe to re-run)
- **Examples**: 
  - `carrier-config.toml.backup-pre-upgrade-40.0`
  - `web-proxy-config.toml.backup-pre-upgrade-40.0`

### WAG Postinstall Timeout

- **Timeout**: 120 seconds (2 minutes) for both carrier and webproxy postinstall scripts
- **Behavior**: If any postinstall script times out or fails:
  - Script output (stdout/stderr) is displayed
  - Component-specific troubleshooting instructions are provided
  - Playbook execution stops for manual intervention
  - User must resolve the issue before continuing


---

## Safety Gates

### Backup Confirmation

Before any upgrade begins, the operator must confirm backups are taken:

```bash
-e privx_backup_confirmed=true
```

If not provided, the playbook will fail immediately with a clear message.

---

### Version-Specific Validation Marker

After upgrading and validating the primary node, the operator must create a
version-specific validation file on the primary node:

```text
/tmp/privx_validation_done-<PRIVX_TARGET_VERSION>
```

Example for version 40.0:

```bash
touch /tmp/privx_validation_done-40.0
```

Secondary nodes will not upgrade unless this file exists on the primary node.

---

## Execution

### Standard PrivX Upgrade Process

#### Phase 1 – Backup validation, RPM validation, stop secondary nodes, and upgrade primary

```bash
ansible-playbook -i inventory --tags primary_upgrade upgrade_privx.yml \
  -e privx_backup_confirmed=true
```

This phase performs:
1. **Backup validation check** (runs once on primary)
2. **RPM availability validation** (checks target version exists on all nodes)
3. **Stops PrivX on ALL secondary nodes** simultaneously
4. **Upgrades PrivX on the primary node**
5. **Displays validation instructions**

---

#### Phase 2 – Manual validation

Validate PrivX functionality on the primary node.

When validation is complete, create the marker file on the **primary node**:

```bash
touch /tmp/privx_validation_done-40.0
```

---

#### Phase 3 – Upgrade secondary nodes

```bash
ansible-playbook -i inventory --tags secondary_upgrade upgrade_privx.yml
```

This phase:
- Verifies the validation marker exists on the primary node
- Upgrades PrivX on all secondary nodes

---

### Zero Downtime Upgrade (ZDU) Process - Alternative

**⚠️ ZDU Requirements:**
- Current PrivX version must be at most **one major version behind** the target
- Valid upgrade paths: `40.x → 41.x`, `41.0 → 41.5`, etc.
- Invalid paths: `39.x → 41.x` (use standard upgrade instead)

#### Single Command ZDU Execution

```bash
ansible-playbook -i inventory upgrade_privx_zdu.yml \
  -e privx_backup_confirmed=true
```

This process performs:
1. **Backup validation check** (mandatory confirmation)
2. **ZDU version compatibility check** (validates upgrade path)
3. **RPM availability validation** (checks target version on all nodes)
4. **Execute upgrade_first_stage.sh** on all nodes (serial execution with 30s wait)
5. **Validate target version** installed on all nodes
6. **Execute upgrade_second_stage.sh** on primary node only
7. **Service verification** and completion confirmation

**Key ZDU Benefits:**
- **No service downtime** during upgrade process
- **Automatic rollback** if first stage fails on any node
- **Built-in validation** at each stage
- **Single command execution** (no manual intervention required)

### Upgrade PrivX-Extender nodes (Optional - Separate Playbook)

**Prerequisites**: After completing PrivX core upgrade, you must:
1. Log into PrivX UI
2. Download new version extender configuration files
3. Make any necessary configuration changes manually
4. Place files in `configuration_files/` directory using naming convention: `<inventory_hostname>-extender-config.toml`

```bash
ansible-playbook -i inventory upgrade_extenders.yml
```

This phase:
- **Validates extender configuration files** exist for each node
- Validates PrivX-Extender RPM availability on the first extender node
- **Backs up existing configuration** (version-specific: `.backup-pre-upgrade-<version>`)
- **Only backs up once per version** (skips if backup already exists)
- Upgrades PrivX-Extender nodes **one by one** to maintain High Availability
- **Conditionally deploys new configuration** (only if RPM upgrade occurred)
- **Conditionally runs postinstall script** (only if RPM upgrade occurred)
- **Includes stabilization wait** between upgrades (only when multiple extenders present)

### Upgrade PrivX Web Access Gateway (Optional - Separate Playbook)

**⚠️ CRITICAL PAIRING REQUIREMENTS:**
- Carrier and WebProxy nodes work in pairs for High Availability
- Nodes must be listed in **identical order** in both inventory groups
- First carrier pairs with first webproxy, second with second, etc.
- **Incorrect pairing will cause service downtime during upgrades**

**Prerequisites**: After completing PrivX core upgrade, you must:
1. Log into PrivX UI
2. Download new version carrier and webproxy configuration files
3. Make any necessary configuration changes manually
4. Place files in `configuration_files/` directory using naming conventions:
   - `<inventory_hostname>-carrier-config.toml`
   - `<inventory_hostname>-web-proxy-config.toml`

```bash
ansible-playbook -i inventory upgrade_wag.yml
```

This phase:
- **Validates WAG configuration files** exist for each carrier and webproxy node
- Validates PrivX-Carrier and PrivX-Web-Proxy RPM availability
- **Validates node pairing** (equal number of carrier and webproxy nodes)
- Upgrades WAG components **in pairs** (Carrier-1 → WebProxy-1 → Carrier-2 → WebProxy-2)
- **Version-specific backups** for both carrier and webproxy configurations
- **Conditional configuration deployment** and postinstall execution
- **Configurable wait times** between pairs

**Pairing Example:**
```ini
[privxcarrier]
carrier-site1.example.com    # Pair 1
carrier-site2.example.com    # Pair 2

[privxwebproxy]  
webproxy-site1.example.com   # Pair 1 (matches carrier-site1)
webproxy-site2.example.com   # Pair 2 (matches carrier-site2)
```

**Note**: WAG components work in pairs and must be upgraded sequentially to maintain High Availability.

---

## Key Improvements

- **Correct sequencing**: Secondary nodes are stopped BEFORE primary upgrade begins
- **Parallel operations**: All secondary nodes are stopped simultaneously
- **Clean structure**: Direct task inclusion without unnecessary role overhead
- **Proper host targeting**: Each stage targets the appropriate host groups
- **Organized task structure**: Tasks grouped by component (upgrade_privx/, upgrade_extender/)
- **Version-specific backups**: Extender configs backed up per version with clear naming
- **Conditional execution**: Config deployment and postinstall only run when needed
- **Smart stabilization**: Wait time only applies when multiple extenders present
- **Idempotent operations**: Safe to re-run without side effects

---

## Idempotency and Re-runs

- The playbook is safe to re-run
- Secondary upgrades cannot proceed without validation marker
- Validation is tied to the target version
- No reliance on inventory ordering

---

## Design Principles

- **Explicit operator intent** required at each stage
- **Manual validation gates** prevent automatic progression
- **No long-running waits** or blocking operations
- **CI/CD friendly** with clear stage separation
- **Safe for production** PrivX clusters
- **Idempotent operations** - safe to re-run multiple times
- **Version-aware backups** - each upgrade version tracked separately
- **Conditional execution** - only perform necessary operations
- **High Availability aware** - respects HA requirements for extenders

---

## Troubleshooting

### Common Issues

1. **Variables not found**: Ensure all required variables are defined in inventory
2. **Secondary nodes not stopping**: Check host group membership and connectivity
3. **Validation file not found**: Ensure the file is created on the primary node with correct path
4. **ZDU version compatibility failure**: Current version is more than one major version behind target
5. **ZDU first stage failure**: Check PrivX logs and system resources on failed nodes
6. **ZDU version validation failure**: First stage didn't properly install target version
7. **Extender postinstall timeout**: Script exceeded 2-minute timeout (check logs and troubleshoot manually)
8. **WAG config files missing**: Download from PrivX UI and place in `configuration_files/` directory
9. **WAG node pairing mismatch**: Ensure equal number of carrier and webproxy nodes in correct order
10. **WAG pairing sequence error**: Verify carrier and webproxy nodes are listed in matching order
11. **WAG postinstall timeout**: Carrier or webproxy postinstall exceeded 2-minute timeout
12. **Template rendering errors**: Ensure `privx_target_version` is properly defined as string or number
13. **Backup already exists**: Normal behavior - version-specific backups prevent duplicates

### Debug Commands

```bash
# Check inventory groups
ansible-inventory -i inventory --list

# Test connectivity
ansible -i inventory all -m ping

# Verify variables
ansible -i inventory all -m debug -a "var=privx_target_version"

# Check WAG configuration files
ls -la configuration_files/*-carrier-config.toml configuration_files/*-web-proxy-config.toml

# Verify WAG backup files
ansible -i inventory privxcarrier -m shell -a "ls -la /opt/privx/etc/carrier-config.toml*"
ansible -i inventory privxwebproxy -m shell -a "ls -la /opt/privx/etc/web-proxy-config.toml*"

# Verify WAG node pairing
ansible-inventory -i inventory --list | jq '.privxcarrier.hosts, .privxwebproxy.hosts'

# Test ZDU version compatibility
ansible -i inventory privx_primary:privx_additional_nodes -m shell -a "rpm -q PrivX --queryformat '%{VERSION}'"

# Check ZDU script availability
ansible -i inventory privx_primary:privx_additional_nodes -m shell -a "ls -la /opt/privx/scripts/upgrade_*_stage.sh"

# Check extender configuration version (V1 vs V2)
grep "extender_mode" configuration_files/*-extender-config.toml

# Verify extender configuration files exist
ls -la configuration_files/*-extender-config.toml

```

---

## Notes

- The playbook does not start the PrivX service explicitly
- Always test upgrades in non-production environments first
- Ensure proper backups before running any upgrade
- Extender configuration files must be downloaded from PrivX UI after core upgrade
- Carrier and WebProxy configuration files must be downloaded from PrivX UI after core upgrade
- Version-specific backups allow for easy rollback if needed
- Configuration deployment and postinstall only run when RPM upgrade occurs
- Stabilization wait only applies when multiple extenders are present

---

## License

[![See LICENSE](https://img.shields.io/github/license/SSHcom/ansible-upgrade-privx.svg?style=for-the-badge)](LICENSE)
