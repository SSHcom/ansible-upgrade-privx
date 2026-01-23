# PrivX Ansible Upgrade Testing Guide

## Automated Testing Pipeline

The repository includes comprehensive automated testing through GitHub Actions with the following test jobs:

### 1. **Linting Tests** (`lint`)
- **ansible-lint**: Validates Ansible best practices and syntax
- **yamllint**: Ensures YAML formatting consistency
- **Configuration**: Uses `.ansible-lint` and `.yamllint` config files
- **Python Version**: 3.9
- **Dependencies**: ansible, ansible-lint, yamllint

### 2. **Validation Tests** (`validate`)
- **Syntax Check**: Validates all playbook syntax without execution (`ansible-playbook --syntax-check`)
- **Inventory Validation**: Ensures inventory structure is valid (`ansible-inventory --list`)
- **Python Version**: 3.9
- **Dependencies**: ansible

### 3. **Inventory Validation** (`inventory-validation`)
- **Graph Generation**: Tests inventory graph generation (`ansible-inventory --graph`)
- **Structure Parsing**: Validates inventory parsing and host group structure
- **Output Validation**: Ensures inventory can be parsed without errors
- **Dependencies**: ansible

### 4. **Security Tests** (`security`)
- **GitLeaks**: Scans for secrets and credentials using custom `.gitleaks.toml`
- **Semgrep**: Security vulnerability scanning with multiple rulesets:
  - `p/security-audit`: General security issues
  - `p/secrets`: Secret detection
  - `p/owasp-top-ten`: OWASP Top 10 vulnerabilities
- **Hardcoded Data Check**: Searches for IP addresses and passwords in YAML files
- **Git History**: Full history scan with `fetch-depth: 0`
- **Error Handling**: Uses `continue-on-error: true` to avoid blocking deployments

## Security Configuration

### GitLeaks Configuration (`.gitleaks.toml`)
- Custom rules for PrivX-specific credentials
- Allowlist for template files to reduce false positives
- Detects:
  - PrivX client IDs and secrets
  - SSH private keys
  - Generic API keys

### Pre-commit Hooks (`.pre-commit-config.yaml`)
- Runs security and quality checks before commits
- Includes GitLeaks, ansible-lint, yamllint
- Install with: `pip install pre-commit && pre-commit install`

## Manual Testing Commands

### Run Individual Tests Locally

```bash
# Install dependencies
pip install ansible ansible-lint yamllint

# Linting tests
ansible-lint
yamllint .

# Syntax validation
ansible-playbook -i inventory --syntax-check *.yml

# Inventory validation
ansible-inventory -i inventory/hosts.ini --list
ansible-inventory -i inventory/hosts.ini --graph

# Security scanning
gitleaks detect --config .gitleaks.toml --verbose --no-color
```

### Test Specific Playbooks

```bash
# Test individual playbook syntax
ansible-playbook -i inventory --syntax-check upgrade_privx.yml
ansible-playbook -i inventory --syntax-check upgrade_privx_zdu.yml
ansible-playbook -i inventory --syntax-check upgrade_extenders.yml
ansible-playbook -i inventory --syntax-check upgrade_wag.yml
```

### Security Testing

```bash
# GitLeaks secret detection
gitleaks detect --config .gitleaks.toml

# Manual hardcoded data check
grep -r --include="*.yml" --include="*.yaml" -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' . | grep -v example
grep -r --include="*.yml" --include="*.yaml" -i "password.*=" . | grep -v example

# Install and run Semgrep locally
pip install semgrep
semgrep --config=p/security-audit --config=p/secrets --config=p/owasp-top-ten .
```

## Test Coverage

The testing pipeline covers:
- ✅ Ansible syntax and best practices (ansible-lint)
- ✅ YAML formatting and structure (yamllint)
- ✅ Playbook syntax validation
- ✅ Inventory configuration and parsing
- ✅ Security vulnerabilities and secrets (GitLeaks + Semgrep)
- ✅ Hardcoded credentials and IP addresses detection

## Continuous Integration

All tests run automatically on:
- **Push** to any branch
- **Pull Request** creation/updates

### Test Job Dependencies
- All jobs run in parallel (no dependencies)
- Security tests use `continue-on-error: true` to avoid blocking deployments
- Full git history is fetched for comprehensive secret scanning

### Supported Platforms
- **Runner**: ubuntu-latest
- **Python**: 3.9
- **Ansible**: Latest stable version

## Troubleshooting

### Common Issues

1. **ansible-lint failures**: Check `.ansible-lint` configuration and fix reported issues
2. **yamllint failures**: Check `.yamllint` configuration for formatting rules
3. **GitLeaks findings**: Review `.gitleaks.toml` allowlist or remove sensitive data
4. **Inventory errors**: Validate `inventory/hosts.ini` syntax and host group structure

### Local Development Setup

```bash
# Install all testing dependencies
pip install ansible ansible-lint yamllint pre-commit

# Set up pre-commit hooks
pre-commit install

# Run all tests locally before pushing
ansible-lint
yamllint .
ansible-playbook -i inventory --syntax-check *.yml
gitleaks detect --config .gitleaks.toml
```