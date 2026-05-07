---
name: troubleshooting
description: Troubleshooting guide for failed Ansible deployments, covering common error patterns, network connectivity issues, SSH problems, privilege escalation failures, and vault decryption errors. Reference this skill when diagnosing playbook failures.
---

# Troubleshooting Failed Deployments

This skill provides structured guidance for diagnosing and resolving common Ansible deployment failures. Each section covers a category of failure, explains how to identify the root cause, and provides resolution steps.

## Analyzing Failed Task Output

When a playbook fails, Ansible provides structured error output. Here is how to read it effectively.

### Understanding the Error Structure

A typical Ansible failure produces output in this format:

```
TASK [role_name : Task description] *******************************************
fatal: [hostname]: FAILED! => {
    "changed": false,
    "msg": "Error message from the module",
    "rc": 1,
    "stderr": "detailed error output from the command",
    "stdout": "standard output if any"
}
```

Key fields to examine:
- `msg` contains the module-level error description.
- `rc` is the return code from the underlying command (non-zero indicates failure).
- `stderr` contains detailed error output that often reveals the root cause.
- `stdout` may contain partial output from before the failure.

### Increasing Verbosity

Run the playbook with increased verbosity to get more diagnostic information:

```bash
# Show task results
ansible-playbook playbook.yml -v

# Show task inputs and results
ansible-playbook playbook.yml -vv

# Show connection debugging
ansible-playbook playbook.yml -vvv

# Show full connection and module debugging
ansible-playbook playbook.yml -vvvv
```

Use `-vvv` when debugging connection issues and `-vv` when debugging module behavior.

### Isolating the Failure

Run only the failing task using tags or the `--start-at-task` option:

```bash
# Start at the failing task
ansible-playbook playbook.yml --start-at-task "Task name that failed"

# Run only tasks with a specific tag
ansible-playbook playbook.yml --tags "configure"

# Limit to the failing host
ansible-playbook playbook.yml --limit "failing-host.example.internal"

# Step through tasks one at a time
ansible-playbook playbook.yml --step
```

## Common Ansible Error Patterns and Fixes

### Module Not Found

**Error:**
```
ERROR! couldn't resolve module/action 'community.general.ufw'. This often indicates a misspelling, missing collection, or incorrect module path.
```

**Cause:** The required Ansible collection is not installed.

**Resolution:**
```bash
# Check installed collections
ansible-galaxy collection list

# Install the missing collection
ansible-galaxy collection install community.general

# Or install all required collections from a requirements file
ansible-galaxy collection install -r collections/requirements.yml
```

### Variable Undefined

**Error:**
```
fatal: [hostname]: FAILED! => {"msg": "The task includes an option with an undefined variable. The error was: 'app_listen_port' is undefined"}
```

**Cause:** A required variable is not defined in any of the variable sources (defaults, vars, group_vars, host_vars, extra vars, or registered variables).

**Resolution:**
1. Check the variable precedence chain. Ansible resolves variables in a specific order - see the Ansible documentation for the full list.
2. Verify the variable is defined in the correct file:
   ```bash
   grep -r "app_listen_port" roles/*/defaults/ roles/*/vars/ group_vars/ host_vars/
   ```
3. If the variable should come from a vault file, verify the vault file is included in `vars_files` and the vault password is provided.
4. Add a default value in `defaults/main.yml` if appropriate, or add a pre-task assertion to fail early with a clear message.

### Permission Denied

**Error:**
```
fatal: [hostname]: FAILED! => {"changed": false, "msg": "Permission denied"}
```

**Cause:** The task requires elevated privileges but `become: true` is not set, or the become method is misconfigured.

**Resolution:**
1. Add `become: true` to the task or block.
2. Verify the remote user has sudo permissions on the target host.
3. Check that the become password is provided if required:
   ```bash
   ansible-playbook playbook.yml --ask-become-pass
   ```
4. If using a custom become method, verify it is configured in `ansible.cfg`:
   ```ini
   [privilege_escalation]
   become_method = sudo
   become_ask_pass = False
   ```

### Template Rendering Errors

**Error:**
```
fatal: [hostname]: FAILED! => {"msg": "AnsibleError: template error while templating string: unexpected '}'"}
```

**Cause:** Jinja2 syntax error in a template file or inline template expression.

**Resolution:**
1. Check the template file for unbalanced braces, missing filters, or incorrect syntax.
2. Validate the template locally:
   ```bash
   python3 -c "
   from jinja2 import Environment
   env = Environment()
   template = env.from_string(open('templates/config.yml.j2').read())
   print('Template syntax is valid')
   "
   ```
3. Common mistakes:
   - Using `{{ variable }}` inside a `when` condition (should be bare `variable`).
   - Missing quotes around Jinja2 expressions that start a YAML value: `key: "{{ value }}"`.
   - Using Python methods that are not available in Jinja2 (e.g., `.append()`).

### Timeout Errors

**Error:**
```
fatal: [hostname]: FAILED! => {"msg": "Timeout (12s) waiting for privilege escalation prompt"}
```

**Cause:** The SSH connection or privilege escalation took longer than the configured timeout.

**Resolution:**
1. Increase the connection timeout in `ansible.cfg`:
   ```ini
   [defaults]
   timeout = 30

   [ssh_connection]
   ssh_args = -o ConnectTimeout=30
   ```
2. For long-running tasks, use `async` and `poll`:
   ```yaml
   - name: Run long-running database migration
     ansible.builtin.command:
       cmd: /opt/app/migrate.sh
     async: 600
     poll: 15
     tags:
       - migrate
   ```
3. If privilege escalation is slow, check whether the target host has DNS resolution issues that delay sudo.

## Network Connectivity Debugging

### Host Unreachable

**Error:**
```
fatal: [hostname]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host hostname port 22: Connection timed out"}
```

**Diagnostic steps:**

1. Verify basic network connectivity from the Ansible control node:
   ```bash
   ping -c 3 hostname
   ```

2. Verify the SSH port is open:
   ```bash
   nc -zv hostname 22 -w 5
   ```

3. Check the inventory for correct hostname or IP:
   ```bash
   ansible-inventory -i inventory/ --host hostname
   ```

4. Test SSH manually:
   ```bash
   ssh -v -i /path/to/key user@hostname
   ```

5. If behind a bastion host, verify the proxy configuration in `ansible.cfg`:
   ```ini
   [ssh_connection]
   ssh_args = -o ProxyCommand="ssh -W %h:%p -q bastion.example.internal"
   ```

### DNS Resolution Failures

**Error:**
```
fatal: [app-server-01]: UNREACHABLE! => {"msg": "Failed to connect to the host via ssh: ssh: Could not resolve hostname app-server-01: Name or service not known"}
```

**Resolution:**
1. Verify DNS resolution: `nslookup app-server-01` or `dig app-server-01`.
2. If using short hostnames, verify the DNS search domain is configured correctly in `/etc/resolv.conf`.
3. As a temporary workaround, use the IP address in the inventory with `ansible_host`:
   ```yaml
   all:
     hosts:
       app-server-01:
         ansible_host: 10.20.30.40
   ```

## SSH Key and Authentication Issues

### Key Authentication Failure

**Error:**
```
fatal: [hostname]: UNREACHABLE! => {"msg": "Failed to connect to the host via ssh: Permission denied (publickey,gssapi-keyex,gssapi-with-mic)"}
```

**Diagnostic steps:**

1. Verify the correct key is being used:
   ```bash
   ssh -v -i /path/to/key user@hostname 2>&1 | grep "Offering"
   ```

2. Check file permissions on the key:
   ```bash
   ls -la /path/to/key
   # Must be 0600 for private key, 0644 for public key
   chmod 600 /path/to/key
   ```

3. Verify the public key is in the target host's `~/.ssh/authorized_keys`.

4. Check the remote user exists and has the correct home directory permissions:
   ```bash
   # On the target host
   ls -la /home/user/.ssh/
   # .ssh directory must be 0700
   # authorized_keys must be 0600
   ```

5. Configure the key in `ansible.cfg` or inventory:
   ```ini
   [defaults]
   private_key_file = /path/to/key
   remote_user = deploy_user
   ```

### Host Key Verification Failure

**Error:**
```
fatal: [hostname]: UNREACHABLE! => {"msg": "Failed to connect to the host via ssh: Host key verification failed."}
```

**Resolution:**
1. Add the host key to the known_hosts file:
   ```bash
   ssh-keyscan -H hostname >> ~/.ssh/known_hosts
   ```
2. If the host was rebuilt and has a new key, remove the old key:
   ```bash
   ssh-keygen -R hostname
   ssh-keyscan -H hostname >> ~/.ssh/known_hosts
   ```
3. Do not disable host key checking in production. The `ANSIBLE_HOST_KEY_CHECKING=False` setting is acceptable only in ephemeral CI/CD environments with disposable infrastructure.

## Privilege Escalation Issues

### Sudo Password Required

**Error:**
```
fatal: [hostname]: FAILED! => {"msg": "Missing sudo password"}
```

**Resolution:**
1. Provide the become password:
   ```bash
   ansible-playbook playbook.yml --ask-become-pass
   ```
2. Or configure passwordless sudo for the Ansible user on target hosts:
   ```
   deploy_user ALL=(ALL) NOPASSWD: ALL
   ```
3. For more restrictive environments, limit passwordless sudo to specific commands:
   ```
   deploy_user ALL=(ALL) NOPASSWD: /usr/bin/systemctl, /usr/bin/yum, /usr/bin/cp
   ```

### Sudo Requiretty

**Error:**
```
fatal: [hostname]: FAILED! => {"msg": "Timeout (12s) waiting for privilege escalation prompt"}
```

**Cause:** The target host has `requiretty` set in `/etc/sudoers`, which prevents sudo from working over non-interactive SSH sessions.

**Resolution:**
1. Remove or comment out the `Defaults requiretty` line in `/etc/sudoers` on the target host.
2. Or add an exception for the Ansible user:
   ```
   Defaults:deploy_user !requiretty
   ```
3. Ansible can be configured to use a pseudo-TTY:
   ```ini
   [ssh_connection]
   ssh_args = -o ControlMaster=auto -o ControlPersist=60s -tt
   ```

## Vault Decryption Failures

### Wrong Vault Password

**Error:**
```
ERROR! Decryption failed (no vault secrets were found that could decrypt)
```

**Resolution:**
1. Verify you are using the correct vault password file:
   ```bash
   ansible-playbook playbook.yml --vault-password-file /path/to/vault_pass
   ```
2. If using multiple vault IDs, specify the correct ID:
   ```bash
   ansible-playbook playbook.yml --vault-id prod@/path/to/prod_vault_pass
   ```
3. Test decryption of a specific file:
   ```bash
   ansible-vault view vars/vault_secrets.yml --vault-password-file /path/to/vault_pass
   ```

### Vault-Encrypted Variable in Wrong Format

**Error:**
```
ERROR! input is not vault encrypted data
```

**Cause:** The file or variable is not properly encrypted, or the file contains a mix of encrypted and unencrypted content.

**Resolution:**
1. Verify the file starts with `$ANSIBLE_VAULT;1.1;AES256`:
   ```bash
   head -1 vars/vault_secrets.yml
   ```
2. If the file is partially encrypted, re-encrypt the entire file:
   ```bash
   ansible-vault encrypt vars/vault_secrets.yml
   ```
3. For inline encrypted variables, ensure the indentation is preserved:
   ```yaml
   vault_db_password: !vault |
     $ANSIBLE_VAULT;1.1;AES256
     61626364656667686970...
   ```

### Vault Password File Permissions

**Error:**
```
ERROR! The vault password file /path/to/vault_pass is not readable
```

**Resolution:**
1. Check file permissions: `ls -la /path/to/vault_pass`.
2. The vault password file should be readable by the user running Ansible and not world-readable:
   ```bash
   chmod 600 /path/to/vault_pass
   ```
3. Verify the file is not empty and does not contain a trailing newline that could corrupt the password.

## General Debugging Workflow

When encountering an unfamiliar error, follow this systematic approach:

1. Read the full error message, including `stderr` output.
2. Increase verbosity with `-vvv` and run the failing task in isolation.
3. Verify connectivity and authentication to the target host manually.
4. Check that all required collections and Python libraries are installed on both the control node and target hosts.
5. Run the playbook in check mode (`--check`) to see if the error is in the execution or the planning phase.
6. Search the team's internal knowledge base and Ansible documentation for the specific error message.
7. If the error is in a custom module or script, test that module or script independently on the target host.
8. Capture the full output with `-vvvv` and include it in any escalation to the team.
