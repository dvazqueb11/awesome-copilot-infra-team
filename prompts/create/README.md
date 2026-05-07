# Prompts for Creating Infrastructure Code

This collection contains ready-to-use prompts for generating new Ansible playbooks, roles, templates, and Python utilities. Copy the prompt text from the fenced block and paste it into a Copilot session. Adjust the bracketed placeholders to match your use case.

---

## Create a new playbook for service deployment

Use this when you need a playbook that deploys and configures a systemd service on RHEL-based servers.

```
Create an Ansible playbook that deploys [service name] to servers in the [host group] inventory group. The playbook should:

- Use fully qualified collection names for all modules
- Include pre-tasks that validate all required variables are defined
- Install the service package from the internal yum repository
- Deploy a configuration file from a Jinja2 template at /etc/[service name]/config.yml
- Ensure the service is enabled and started
- Include a post-task that verifies the service health endpoint at http://localhost:[port]/health
- Wrap state-changing tasks in block/rescue/always with proper error handling
- Tag every task with appropriate tags: install, configure, verify
- Use handlers for service restarts instead of inline restart tasks
- Follow our variable naming convention: prefix all variables with the service name in snake_case
- Never hardcode credentials - use vault variable references

Required variables to define:
- [service]_package_version
- [service]_listen_port
- [service]_config_options (dictionary)
- vault_[service]_api_key
```

---

## Create a Jinja2 configuration template

Use this when you need a templated configuration file for a service managed by Ansible.

```
Create a Jinja2 template (.j2 file) for [service name] configuration. The template should generate a [YAML/TOML/INI] configuration file with these sections:

- Server settings: listen address, port, TLS configuration
- Database connection: host, port, database name, credentials from vault variables
- Logging: log level, log format, log file path, rotation settings
- Application-specific settings: [describe your settings]

Requirements:
- Include a header comment documenting the template's purpose, expected variables, and that the file is managed by Ansible
- Use default filters for optional variables: {{ variable | default('fallback') }}
- Use conditional blocks for optional sections that should only appear when enabled
- Validate that required variables are of the expected type using Jinja2 tests where appropriate
- Quote all string values that could be interpreted as YAML booleans or numbers
- Follow our naming convention: all template variables should be prefixed with [role_name]_
```

---

## Create a new Ansible role from scratch

Use this when starting a new role for a service or capability that does not have an existing role.

```
Create a complete Ansible role named [role_name] that [describe what the role does]. Generate all standard role directories and files:

1. defaults/main.yml - All configurable variables with descriptive comments and sensible defaults. Prefix every variable with [role_name]_.
2. vars/main.yml - Internal role variables that should not be overridden.
3. tasks/main.yml - Entry point that includes task files for install, configure, and verify phases.
4. tasks/install.yml - Package installation with block/rescue error handling.
5. tasks/configure.yml - Configuration deployment with template tasks and handler notifications.
6. tasks/verify.yml - Post-configuration verification that the service is running and healthy.
7. handlers/main.yml - Handlers for restart and reload, prefixed with "[role_name] - ".
8. templates/ - Any configuration templates needed, using .j2 extension.
9. meta/main.yml - Role metadata with supported platforms (RHEL 8, RHEL 9, Ubuntu 22.04) and dependencies.
10. molecule/default/ - Molecule test scaffolding with molecule.yml, converge.yml, and verify.yml.
11. README.md - Role documentation with description, variables, dependencies, and example playbook.

Use FQCN for all modules. Tag every task. Ensure idempotency.
```

---

## Create a dynamic inventory script in Python

Use this when you need a custom inventory source that queries an internal CMDB or cloud API.

```
Create a Python dynamic inventory script for Ansible that queries [data source - e.g., ServiceNow CMDB, internal API, cloud provider]. The script should:

- Accept --list and --host arguments per the Ansible dynamic inventory specification
- Return JSON in the format Ansible expects: groups with hosts, vars, and children
- Include type hints on all function signatures
- Use structlog for logging
- Load configuration from environment variables: [API_URL], [API_TOKEN]
- Handle API errors gracefully with retries (3 attempts with exponential backoff)
- Cache results locally in a JSON file with a configurable TTL (default 300 seconds) to avoid excessive API calls
- Group hosts by: environment (production, staging, development), operating system, data center location, and application tier
- Include host variables: ansible_host (IP), ansible_user, datacenter, environment, application_name
- Include a main block that supports being called directly for testing
- Follow PEP 8 and include a module docstring explaining usage

Example output structure:
{
  "production": {"hosts": ["app-01.example.internal"], "vars": {}},
  "_meta": {"hostvars": {"app-01.example.internal": {"ansible_host": "10.0.1.10"}}}
}
```

---

## Create a playbook for certificate renewal

Use this to generate a playbook that handles TLS certificate renewal across a fleet of servers.

```
Create an Ansible playbook that renews TLS certificates on servers in the [host group] group. The playbook should:

- Run serially (one host at a time) to avoid service disruption
- Check current certificate expiry date before proceeding
- Only renew certificates expiring within [N] days
- Generate a new CSR using the existing private key
- Submit the CSR to the internal CA at [CA endpoint] using the ansible.builtin.uri module
- Deploy the new certificate and any intermediate CA certificates
- Verify the certificate chain is valid using openssl
- Reload the service that uses the certificate (do not restart to avoid connection drops)
- Verify the service is responding with the new certificate
- Include block/rescue that restores the previous certificate if renewal fails
- Log all renewal activity to /var/log/cert_renewal.log
- Send a summary notification on completion with the count of renewed and skipped certificates

Variables needed:
- cert_renewal_threshold_days (default: 30)
- cert_private_key_path
- cert_csr_subject
- vault_ca_api_token
```

---

## Create a Python utility for log analysis

Use this when you need a script that parses and analyzes application or system logs.

```
Create a Python CLI utility that analyzes [log type - e.g., Ansible playbook output, application logs, syslog] and produces a structured summary report. The script should:

- Use click for CLI argument parsing with --input-file, --output-format (json, table, csv), and --since (time filter) options
- Parse log entries into structured records with timestamp, level, source, and message fields
- Calculate summary statistics: total entries, entries by severity, error rate, most frequent error messages
- Identify patterns: repeated errors within a time window, correlated failures across hosts
- Output results in the requested format
- Include type hints on all functions
- Use structlog for the script's own logging
- Handle malformed log lines gracefully (log a warning and skip)
- Include unit tests for the parsing and statistics functions
- Follow PEP 8 with a maximum line length of 100 characters
```

---

## Create an inventory file for a multi-environment setup

Use this when setting up inventory for a new project with multiple environments.

```
Create a YAML inventory file structure for a project with the following environments: production, staging, and development. Each environment has:

- Web servers: [N] hosts per environment
- Application servers: [N] hosts per environment
- Database servers: [N] hosts per environment (primary + replica)

Requirements:
- Use the YAML inventory format (not INI)
- Organize as: inventory/[environment]/hosts.yml and inventory/[environment]/group_vars/all.yml
- Define group hierarchy: all > [environment] > [tier] (e.g., production > production_webservers)
- Include group variables for each environment: dns_suffix, ntp_servers, yum_repo_base_url, log_server
- Include host-level variables where hosts differ: ansible_host (IP address), disk configuration
- Use realistic but anonymized hostnames following the pattern: [role]-[env]-[seq].example.internal
- Add comments explaining the inventory structure for new team members
```

---

## Create a playbook for automated patching

Use this to generate a patching playbook that applies OS updates with proper controls.

```
Create an Ansible playbook for applying operating system patches to RHEL-based servers. The playbook should:

- Run against hosts in the [host group] group with serial: "20%" to limit blast radius
- Take a pre-patch snapshot of the system state (installed packages, running services, disk usage)
- Check for and apply security patches only (not all updates) using ansible.builtin.yum with security=true
- Register which packages were updated
- Restart services that depend on updated packages (use needs-restarting to identify them)
- Run a post-patch health verification that checks:
  - All previously running services are still running
  - No new listening ports have appeared
  - Application health endpoints respond with 200
  - Disk usage has not exceeded thresholds
- If health verification fails, use block/rescue to:
  - Roll back the updated packages using yum history undo
  - Verify the rollback succeeded
  - Fail the host with a clear error message
- Generate a patch report as a JSON file with: hostname, packages updated, services restarted, health check result
- Support --check mode for dry-run patch assessment
- Tag all tasks: patch, verify, rollback, report
```

---

## Create a Python wrapper for Ansible vault operations

Use this when you need a CLI tool to manage vault-encrypted secrets across multiple files and environments.

```
Create a Python CLI tool that simplifies Ansible vault operations across the project. The tool should:

- Use click with subcommands: encrypt, decrypt, edit, rotate, audit
- encrypt: Encrypt a specific variable value and output the encrypted string for pasting into a vars file
- decrypt: Decrypt and display a specific variable from a vault file (for debugging)
- edit: Open a vault file in the default editor with automatic re-encryption on save
- rotate: Re-encrypt all vault files with a new password (for credential rotation)
- audit: Scan the project for plaintext secrets that should be encrypted (API keys, passwords, tokens)
- Support multiple vault IDs for per-environment encryption
- Load vault passwords from files, environment variables, or interactive prompt
- Include type hints on all functions
- Log all operations with structlog (never log decrypted values)
- Include unit tests for the audit detection patterns
```
