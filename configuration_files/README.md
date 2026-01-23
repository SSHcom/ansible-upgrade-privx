# PrivX Component Configuration Files

This directory contains the configuration files required for upgrading PrivX additional components that require configuration management.

## Supported Components

### PrivX Extender
- **Package**: PrivX-Extender
- **Configuration file**: `extender-config.toml`
- **Playbook**: `upgrade_extenders.yml`

### PrivX Carrier (Web Access Gateway)
- **Package**: PrivX-Carrier
- **Configuration file**: `carrier-config.toml`
- **Playbook**: `upgrade_wag.yml`

### PrivX WebProxy (Web Access Gateway)
- **Package**: PrivX-Web-Proxy
- **Configuration file**: `web-proxy-config.toml`
- **Playbook**: `upgrade_wag.yml`

## File Naming Conventions

Configuration files must follow these naming patterns:

### PrivX Extender
```
<ansible_inventory_hostname>-extender-config.toml
```

### PrivX Carrier
```
<ansible_inventory_hostname>-carrier-config.toml
```

### PrivX WebProxy
```
<ansible_inventory_hostname>-webproxy-config.toml
```

## Process

1. **After PrivX core upgrade completion**, log into PrivX UI
2. **Download** the new version configuration files for each component node
3. **Make any necessary configuration changes** manually
4. **Save each file** in this directory using the correct naming convention

## Examples

If your inventory has these nodes:
- **Extender**: `extender-node1.example.com`, `extender-node2.example.com`
- **Carrier**: `carrier-node1.example.com`, `carrier-node2.example.com`
- **WebProxy**: `webproxy-node1.example.com`, `webproxy-node2.example.com`

The configuration files should be named:

### Extender Configuration Files:
- `configuration_files/extender-node1.example.com-extender-config.toml`
- `configuration_files/extender-node2.example.com-extender-config.toml`

### Carrier Configuration Files:
- `configuration_files/carrier-node1.example.com-carrier-config.toml`
- `configuration_files/carrier-node2.example.com-carrier-config.toml`

### WebProxy Configuration Files:
- `configuration_files/webproxy-node1.example.com-webproxy-config.toml`
- `configuration_files/webproxy-node2.example.com-webproxy-config.toml`

## Component-Specific Requirements

### PrivX Extender
- **Upgrade sequence**: One by one for High Availability
- **Configuration source**: PrivX UI → Extender configuration
- **Backup location**: `/opt/privx/etc/extender-config.toml.backup-pre-upgrade-<version>`

### PrivX Web Access Gateway (Carrier + WebProxy)
- **Upgrade sequence**: Paired upgrades (Carrier → WebProxy for each pair)
- **Configuration source**: PrivX UI → Carrier/WebProxy configuration
- **Backup locations**: 
  - `/opt/privx/etc/carrier-config.toml.backup-pre-upgrade-<version>`
  - `/opt/privx/etc/web-proxy-config.toml.backup-pre-upgrade-<version>`
- **Pairing requirement**: Equal number of carrier and webproxy nodes

## Important Notes

- Configuration files are **required** before running component upgrades
- Files must be downloaded from PrivX UI **after** core PrivX upgrade
- The upgrade playbooks will validate these files exist before proceeding
- **Version-specific backups** are created automatically during upgrades
- Configuration deployment only occurs when RPM upgrades are performed

## Validation Commands

Check if all required configuration files exist:

```bash
# Check extender config files
ls -la configuration_files/*-extender-config.toml

# Check carrier config files  
ls -la configuration_files/*-carrier-config.toml

# Check webproxy config files
ls -la configuration_files/*-webproxy-config.toml

# Check all component config files
ls -la configuration_files/*.toml
```

## Troubleshooting

### Missing Configuration Files
If upgrade playbooks fail with configuration file errors:
1. Verify the file exists in this directory
2. Check the naming convention matches exactly
3. Ensure the file was downloaded after PrivX core upgrade
4. Verify file permissions are readable

### File Naming Issues
- Use the exact inventory hostname (case-sensitive)
- Include the component type in the filename
- Use `.toml` extension
- No spaces or special characters in filenames