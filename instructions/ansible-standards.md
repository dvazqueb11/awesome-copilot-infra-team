# Ansible Coding Standards

This document defines the Ansible coding standards for the HSBC Infrastructure team. All playbooks, roles, and task files must follow these standards. The team's Copilot agents reference this document during code reviews and generation.

## File Naming Conventions

### Playbooks

- Use lowercase with hyphens as separators: `deploy-application.yml`, `rotate-credentials.yml`.
- Name playbooks after the action they perform, not the system they target: `deploy-webserver.yml` instead of `webserver.yml`.
- Prefix playbooks with the action category when the project contains many playbooks:
  - `deploy-` for deployment playbooks
  - `configure-` for configuration-only playbooks
  - `verify-` for verification and audit playbooks
  - `rollback-` for rollback playbooks
  - `maintenance-` for scheduled maintenance tasks

### Roles

- Use lowercase with hyphens as separators: `web-server`, `database-primary`, `monitoring-agent`.
- Name roles after the capability they provide, not the package they install: `application-server` instead of `tomcat`.

### Task Files

- The entry point is always `tasks/main.yml`.
- Split task files by phase: `install.yml`, `configure.yml`, `verify.yml`, `cleanup.yml`.
- Additional task files use descriptive names: `configure-tls.yml`, `configure-logging.yml`.

### Variable Files

- `defaults/main.yml` for variables with sensible defaults that users can override.
- `vars/main.yml` for internal variables that should not be overridden.
- Vault-encrypted files follow the pattern: `vault_<purpose>.yml` (e.g., `vault_database.yml`, `vault_api_keys.yml`).

### Template Files

- Use the `.j2` extension for all Jinja2 templates.
- Name templates after the target file they generate: `nginx.conf.j2`, `application.yml.j2`.
- Place templates in the `templates/` directory within a role.

## Variable Precedence and Naming

### Naming Rules

All variables must use `snake_case`. No exceptions.

Role-specific variables must be prefixed with the role name to prevent collisions when multiple roles are applied to the same host:

```yaml
# Role: web-server
web_server_listen_port: 8080
web_server_document_root: /var/www/html
web_server_enable_tls: true
web_server_tls_cert_path: /etc/pki/tls/certs/server.crt
web_server_allowed_cidrs:
  - 10.0.0.0/8
  - 172.16.0.0/12
```

### Type-Specific Prefixes

- Boolean variables: use `is_` or `enable_` prefix: `web_server_enable_tls`, `database_is_clustered`.
- List variables: use plural names: `web_server_allowed_cidrs`, `monitoring_agent_targets`.
- Dictionary variables: use a descriptive name indicating structure: `web_server_tls_config`, `database_connection_params`.

### Documentation

Every variable in `defaults/main.yml` must have a comment block above it with:
- A description of the variable's purpose
- The expected type (string, integer, boolean, list, dictionary)
- The acceptable range of values or format

```yaml
---
# Port the web server listens on for HTTP traffic.
# Type: integer
# Range: 1024-65535
web_server_listen_port: 8080

# List of CIDR ranges allowed to access the web server.
# Type: list of strings
# Format: CIDR notation (e.g., 10.0.0.0/8)
web_server_allowed_cidrs:
  - 10.0.0.0/8
```

### Precedence Awareness

Be deliberate about where you define variables. The standard locations in order of increasing precedence:

1. `defaults/main.yml` - Lowest precedence. Safe defaults that work for most cases.
2. `group_vars/all.yml` - Variables shared across all hosts in an environment.
3. `group_vars/<group>.yml` - Variables specific to a host group.
4. `host_vars/<host>.yml` - Variables specific to a single host.
5. `vars/main.yml` - Role-internal variables (not intended to be overridden).
6. Extra vars (`-e` flag) - Highest precedence. Use for one-time overrides.

Never define the same variable in multiple files at the same precedence level.

## Task Structure Requirements

### Naming

Every task must have a `name` field that describes the business intent:

```yaml
# Correct: explains what and why
- name: Ensure application configuration reflects current environment settings
  ansible.builtin.template:
    src: app_config.yml.j2
    dest: /etc/app/config.yml

# Incorrect: describes the technical action
- name: Copy file
  ansible.builtin.template:
    src: app_config.yml.j2
    dest: /etc/app/config.yml
```

### FQCN

All module references must use the fully qualified collection name. Short names are not permitted:

```yaml
# Correct
- name: Install required packages
  ansible.builtin.yum:
    name: "{{ packages }}"
    state: present

# Incorrect
- name: Install required packages
  yum:
    name: "{{ packages }}"
    state: present
```

### Tags

Every task must have at least one tag. Use consistent tag names across the project:

- `install` - Package installation tasks
- `configure` - Configuration file deployment and service configuration
- `verify` - Health checks and state verification
- `security` - Security hardening tasks
- `cleanup` - Temporary file removal and resource cleanup
- Compound tags combining the role name and action: `web-server-configure`, `database-backup`

### Error Handling

Tasks that modify system state must be wrapped in `block/rescue/always`:

```yaml
- name: Deploy and configure the application
  tags:
    - configure
  block:
    - name: Deploy application configuration file
      ansible.builtin.template:
        src: app_config.yml.j2
        dest: /etc/app/config.yml
        owner: app_user
        group: app_group
        mode: "0640"
      notify:
        - web-server - reload application

  rescue:
    - name: Log configuration deployment failure
      ansible.builtin.debug:
        msg: "Configuration deployment failed on {{ inventory_hostname }}"

    - name: Fail with actionable error message
      ansible.builtin.fail:
        msg: "Configuration deployment failed. Check template variables and file permissions."
```

### Idempotency

All tasks must be idempotent. Running a playbook twice with no input changes must produce zero changed tasks on the second run.

For `ansible.builtin.shell` and `ansible.builtin.command` tasks, always include at least one of:
- `creates` parameter
- `removes` parameter
- `changed_when` condition
- A `stat` pre-check with a `when` condition

```yaml
- name: Initialize the application database schema
  ansible.builtin.command:
    cmd: /opt/app/bin/init-schema.sh
    creates: /opt/app/.schema_initialized
  tags:
    - install
    - configure
```

### Handlers

Handler names must be unique across the entire project. Prefix handler names with the role name:

```yaml
handlers:
  - name: web-server - restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
    become: true

  - name: web-server - reload nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
    become: true
```

## Role Dependencies Policy

- Minimize role dependencies. Prefer composing roles in playbooks over declaring hard dependencies in `meta/main.yml`.
- When a dependency is necessary, pin it to a specific version or source in `meta/main.yml`.
- Document why the dependency exists with a comment in the meta file.
- Circular dependencies are never acceptable.
- If two roles share common tasks, extract the shared tasks into a dedicated role rather than making one role depend on the other.

## Testing Requirements

### Molecule Tests

Every role must have molecule tests in the `molecule/default/` directory.

Required files:
- `molecule.yml` - Test configuration using the Docker driver with a Rocky Linux 8 image.
- `converge.yml` - A playbook that applies the role with representative variables.
- `verify.yml` - A playbook that asserts the expected state after convergence.

The verify playbook must test observable outcomes:
- Required packages are installed
- Configuration files exist with correct permissions
- Services are running and enabled
- Listening ports match expectations
- Health endpoints respond correctly

### Running Tests

Run the full test cycle:

```bash
molecule test
```

Run just the converge and verify phases during development:

```bash
molecule converge && molecule verify
```

### Lint

All code must pass `ansible-lint` and `yamllint` before merging. Configuration files for both tools must be present in the project root.

## Git Commit Message Format

Use the following format for commit messages related to playbook and role changes:

```
[component] Short description of the change

Longer description explaining why the change was made, what it affects,
and any important context for reviewers.

Affected roles: role-name-1, role-name-2
Affected playbooks: deploy-application.yml
Tested with: molecule test (role-name-1), staging deployment
Change-Record: CHG0012345
```

The `[component]` prefix should be one of:
- `[role:name]` for changes to a specific role
- `[playbook:name]` for changes to a specific playbook
- `[inventory]` for inventory changes
- `[infra]` for project-wide infrastructure changes (ansible.cfg, requirements, CI)

Include the `Change-Record` line when the change is associated with a change management ticket.
