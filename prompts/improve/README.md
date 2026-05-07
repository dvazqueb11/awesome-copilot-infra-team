# Prompts for Improving Existing Infrastructure Code

This collection contains prompts for enhancing, refactoring, and hardening existing Ansible playbooks, roles, and Python scripts. Copy the prompt text from the fenced block and paste it into a Copilot session after opening or referencing the file you want to improve.

---

## Add error handling to an existing playbook

Use this when a playbook has tasks that modify system state but lacks block/rescue/always constructs.

```
Review this playbook and add error handling using block/rescue/always around every group of tasks that modifies system state (installing packages, deploying configuration, restarting services). For each block:

- The rescue section should:
  - Log a debug message with the specific task that failed and the error details
  - Attempt to revert the change if possible (restore previous config, downgrade package)
  - Fail with a clear, actionable error message

- The always section should:
  - Clean up any temporary files created during the block
  - Release any locks acquired during the block

Do not modify tasks that only gather information or run checks. Preserve existing task names, tags, and variable references. Use FQCN for any new modules you add.
```

---

## Optimize a slow playbook

Use this when a playbook takes too long to execute and you need to improve its performance.

```
Analyze this playbook for performance issues and optimize it. Look for these common bottlenecks:

1. Tasks that run sequentially but could use async/poll for parallelism
2. Repeated gather_facts across multiple plays (set gather_facts: false on subsequent plays and use cached facts)
3. Package installation tasks that install one package at a time instead of passing a list
4. Tasks that use shell/command to check state instead of using a purpose-built module
5. Missing free strategy where task independence allows it
6. Loops that could use batch operations (e.g., installing multiple packages in one yum call)
7. Template tasks that trigger handlers unnecessarily (add a when condition or use a checksum comparison)
8. DNS lookups or API calls that could be cached in a set_fact

For each optimization:
- Explain what the current performance issue is
- Show the before and after code
- Estimate the time savings if possible

Do not change the functional behavior of the playbook. Preserve all existing task names and tags.
```

---

## Add structured logging and metrics collection

Use this when you need to add operational visibility to a playbook or Python script.

```
Add structured logging and metrics collection to this code.

For Ansible playbooks:
- Add a pre-task that records the start timestamp and play parameters to a JSON log file at /var/log/ansible/[playbook_name].json
- Add a post-task that records the completion timestamp, play recap (ok/changed/failed counts), and duration
- Add callback plugin configuration in ansible.cfg for the json log callback
- On failure, include the failed task name and error message in the log entry

For Python scripts:
- Replace any print() statements with structlog calls at the appropriate level (info, warning, error)
- Add structured context to every log call: function name, relevant parameters, duration for timed operations
- Add timing metrics for external calls (API requests, file operations, SSH connections)
- Configure structlog with JSON output for machine parsing and human-readable console output for development
- Log entry and exit of major functions at debug level
- Log errors with full context including the exception type, message, and relevant parameters

Do not add logging that includes sensitive data (passwords, tokens, certificate contents).
```

---

## Refactor a monolithic playbook into roles

Use this when a single playbook file has grown too large and needs to be split into reusable roles.

```
Refactor this playbook into a set of Ansible roles. Follow this process:

1. Identify logical groups of tasks that represent a single responsibility (e.g., package installation, service configuration, security hardening, monitoring setup).
2. For each group, create a role with the standard directory structure:
   - defaults/main.yml with all variables that have sensible defaults, prefixed by role name
   - vars/main.yml for internal variables
   - tasks/main.yml that includes phase-specific task files (install.yml, configure.yml, verify.yml)
   - handlers/main.yml with role-name-prefixed handler names
   - templates/ for any Jinja2 templates
   - meta/main.yml with role metadata

3. Create a new site.yml playbook that composes the roles in the correct order.
4. Move all variables to the appropriate role defaults or group_vars.
5. Ensure all inter-role dependencies are documented in meta/main.yml.

Preserve all existing task names and tags. Every module reference must use FQCN. The refactored playbook must produce the same result as the original when run against the same inventory.
```

---

## Add molecule tests to an existing role

Use this when a role exists but has no automated tests.

```
Add molecule test scaffolding and test cases to this Ansible role. Create:

1. molecule/default/molecule.yml - Configuration using the Docker driver with a Rocky Linux 8 image. Include lint configuration for ansible-lint and yamllint.

2. molecule/default/converge.yml - A playbook that applies the role with representative variable values that exercise the main code paths.

3. molecule/default/verify.yml - A verification playbook that asserts:
   - All expected packages are installed
   - All configuration files exist with correct permissions and ownership
   - All expected services are running and enabled
   - Configuration file contents match expected values (spot-check key settings)
   - Health endpoints respond correctly
   - No unexpected error patterns in service logs

4. molecule/default/prepare.yml - A preparation playbook that sets up any prerequisites the role needs (create users, install dependencies, set up mock services).

5. Add any required test variables in molecule/default/group_vars/all.yml.

The tests should be meaningful - they should catch real regressions, not just verify that Ansible ran without errors. Focus on testing the observable behavior (files created, services running, ports listening) rather than implementation details (specific task counts).
```

---

## Security hardening of existing playbooks

Use this when you need to audit and fix security issues in existing Ansible code.

```
Review this code for security issues and apply fixes. Check for and remediate:

1. Hardcoded secrets: Replace any plaintext passwords, API keys, tokens, or certificates with references to Ansible vault variables. Create a vault variable file listing what needs to be encrypted.

2. Insecure file permissions: Change any task that creates files with mode 0644 or more permissive where the file contains sensitive data. Configuration files with credentials should be 0600 or 0640.

3. Unnecessary privilege escalation: Move become: true from the play level to individual tasks that actually require it. Document why each task needs elevated privileges.

4. Shell injection risks: Replace shell/command tasks that interpolate variables with purpose-built modules where available. Where shell is necessary, use pipes and quote variables properly.

5. Missing TLS verification: Ensure all uri module calls use validate_certs: true. Add TLS certificate verification to any API calls.

6. Overly permissive firewall rules: Flag any rules that allow 0.0.0.0/0 or ::/0 access to non-public services.

7. Missing audit logging: Add tasks that log security-relevant actions (user creation, permission changes, firewall modifications) to the audit log.

For each fix, explain the security risk that existed and what the remediation addresses.
```

---

## Add idempotency guards to shell and command tasks

Use this when a playbook uses shell or command modules without proper idempotency controls.

```
Review all ansible.builtin.shell and ansible.builtin.command tasks in this code and add idempotency guards. For each task:

1. Determine what the task is trying to achieve and whether a purpose-built Ansible module exists that would be inherently idempotent. If so, replace the shell/command task with the module.

2. If no module exists, add one of these idempotency patterns:
   - creates: parameter if the task creates a file
   - removes: parameter if the task removes a file
   - A stat pre-check followed by a when condition
   - A changed_when condition that accurately reflects whether the task made a change
   - A register + when pattern that checks current state before acting

3. For every remaining shell/command task, add:
   - changed_when: with a condition that detects actual changes
   - failed_when: with conditions that distinguish real failures from expected non-zero exit codes

4. Add a comment above each shell/command task explaining why a module could not be used.

Running the playbook twice consecutively should produce zero changed tasks on the second run.
```

---

## Convert a playbook to support check mode

Use this when a playbook needs to support dry-run execution for change review.

```
Modify this playbook to fully support Ansible check mode (--check). For each task:

1. Ensure tasks that use modules with native check mode support work correctly in check mode without modification.

2. For shell/command tasks, add check_mode: false if the task gathers information needed by subsequent tasks, or add a conditional that provides mock data in check mode.

3. For tasks that register variables used by later tasks, ensure the registered variable is available in check mode by adding a set_fact fallback.

4. Add a pre-task that detects check mode and logs a message indicating this is a dry run.

5. For tasks that call external APIs or make irreversible changes, add when: not ansible_check_mode to skip them in check mode, and add a debug task that logs what would have been done.

Test the playbook in both check mode and normal mode to verify it completes without errors in both cases.
```

---

## Add retry logic to unreliable tasks

Use this when tasks that depend on external services (APIs, package repositories, DNS) fail intermittently.

```
Identify tasks in this playbook that depend on external services and add appropriate retry logic. Apply these patterns:

1. For ansible.builtin.uri tasks (API calls):
   - Add retries: 5 and delay: 10 with until: condition on the expected status code
   - Add timeout: 30 to prevent hanging connections

2. For package installation tasks (yum, apt):
   - Add retries: 3 and delay: 30 to handle transient repository connectivity issues
   - Add a pre-check that verifies repository reachability

3. For service health checks:
   - Add retries: 10 and delay: 5 with until: condition that checks for healthy response
   - This allows time for services to fully start

4. For file download tasks (get_url):
   - Add retries: 3 and delay: 15
   - Add checksum verification to detect partial downloads
   - Add timeout: 120 for large files

5. For tasks that register results checked by later tasks:
   - Ensure the retry loop properly populates the registered variable on success
   - Add a final assertion after the retry block to fail clearly if all retries were exhausted

Document the retry parameters with comments explaining the chosen values.
```

---

## Standardize variable names across a project

Use this when variable naming is inconsistent across roles and playbooks.

```
Audit all variable definitions and references in this project and standardize the naming. Apply these rules:

1. All variables must use snake_case.
2. Role-specific variables must be prefixed with the role name: [role_name]_variable_name.
3. Boolean variables must use is_ or enable_ prefixes: webserver_enable_tls, database_is_clustered.
4. List variables must use plural names: webserver_allowed_cidrs, not webserver_allowed_cidr.
5. Dictionary variables must have a descriptive name indicating their structure: webserver_tls_config, not webserver_tls.
6. Variables in defaults/main.yml must have a comment explaining their purpose, type, and acceptable values.
7. Remove any unused variables.
8. Consolidate duplicate variables that represent the same value under different names.

For each rename:
- List the old name and new name
- Update all references across playbooks, roles, templates, and inventory files
- Verify no reference is missed by searching the entire project
```
