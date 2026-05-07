---
name: hardening-checklist
description: Security hardening patterns and checklists for Linux servers based on CIS benchmarks and financial services regulatory requirements. Reference this skill when hardening servers, reviewing security configurations, or auditing system settings.
---

# Security Hardening Checklist

This skill provides reference patterns for hardening Linux servers in compliance with CIS benchmarks and the organization's security policies. Each section includes the check to perform, the expected configuration, and an Ansible task pattern to enforce it.

## CIS Benchmark Checks for Linux Servers

### Filesystem Configuration

Ensure `/tmp` is a separate partition with restrictive mount options. Shared memory (`/dev/shm`) must be mounted with `noexec`, `nosuid`, and `nodev`.

```yaml
- name: Verify /tmp partition mount options meet CIS requirements
  tags:
    - security
    - cis
    - filesystem
  block:
    - name: Ensure /tmp is mounted with nodev, nosuid, noexec
      ansible.posix.mount:
        path: /tmp
        src: "{{ hardening_tmp_device }}"
        fstype: "{{ hardening_tmp_fstype | default('ext4') }}"
        opts: "defaults,nodev,nosuid,noexec"
        state: mounted
      become: true

    - name: Ensure /dev/shm is mounted with nodev, nosuid, noexec
      ansible.posix.mount:
        path: /dev/shm
        src: tmpfs
        fstype: tmpfs
        opts: "defaults,nodev,nosuid,noexec"
        state: mounted
      become: true

    - name: Disable automounting of removable media
      ansible.builtin.service:
        name: autofs
        state: stopped
        enabled: false
      become: true
      ignore_errors: true
```

### Core Dumps and Address Space Layout Randomization

```yaml
- name: Configure core dumps and ASLR per CIS benchmark
  tags:
    - security
    - cis
    - kernel
  block:
    - name: Ensure core dumps are restricted
      ansible.builtin.lineinfile:
        path: /etc/security/limits.conf
        regexp: '^\*\s+hard\s+core'
        line: "* hard core 0"
        state: present
      become: true

    - name: Ensure ASLR is enabled
      ansible.posix.sysctl:
        name: kernel.randomize_va_space
        value: "2"
        state: present
        sysctl_set: true
        reload: true
      become: true
```

### Mandatory Access Control

```yaml
- name: Ensure SELinux is enforcing on RHEL-based systems
  tags:
    - security
    - cis
    - mac
  block:
    - name: Set SELinux to enforcing mode
      ansible.posix.selinux:
        policy: targeted
        state: enforcing
      become: true
      when: ansible_os_family == "RedHat"

    - name: Verify SELinux is not disabled in bootloader
      ansible.builtin.lineinfile:
        path: /etc/default/grub
        regexp: 'selinux=0'
        state: absent
      become: true
      check_mode: true
      register: selinux_grub_check
      when: ansible_os_family == "RedHat"

    - name: Flag if SELinux is disabled in bootloader
      ansible.builtin.assert:
        that:
          - selinux_grub_check is not changed
        fail_msg: "SELinux is disabled in GRUB configuration. Remove selinux=0 from GRUB_CMDLINE_LINUX."
        success_msg: "SELinux is not disabled in bootloader configuration"
      when: ansible_os_family == "RedHat"
```

## SSH Hardening

Secure the SSH daemon configuration to prevent unauthorized access and reduce the attack surface.

```yaml
- name: Harden SSH daemon configuration
  tags:
    - security
    - ssh
  block:
    - name: Deploy hardened sshd configuration
      ansible.builtin.template:
        src: sshd_config.j2
        dest: /etc/ssh/sshd_config
        owner: root
        group: root
        mode: "0600"
        validate: "/usr/sbin/sshd -t -f %s"
      become: true
      notify:
        - hardening - restart sshd

    - name: Ensure SSH root login is disabled
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: "PermitRootLogin no"
        state: present
      become: true
      notify:
        - hardening - restart sshd

    - name: Ensure SSH password authentication is disabled
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: "PasswordAuthentication no"
        state: present
      become: true
      notify:
        - hardening - restart sshd

    - name: Set SSH MaxAuthTries to 3
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?MaxAuthTries'
        line: "MaxAuthTries 3"
        state: present
      become: true
      notify:
        - hardening - restart sshd

    - name: Set SSH idle timeout interval
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?ClientAliveInterval'
        line: "ClientAliveInterval 300"
        state: present
      become: true
      notify:
        - hardening - restart sshd

    - name: Set SSH client alive count max
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?ClientAliveCountMax'
        line: "ClientAliveCountMax 0"
        state: present
      become: true
      notify:
        - hardening - restart sshd

    - name: Restrict SSH to approved ciphers
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?Ciphers'
        line: "Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr"
        state: present
      become: true
      notify:
        - hardening - restart sshd

    - name: Restrict SSH to approved MACs
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?MACs'
        line: "MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256"
        state: present
      become: true
      notify:
        - hardening - restart sshd

    - name: Restrict SSH to approved key exchange algorithms
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?KexAlgorithms'
        line: "KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group-exchange-sha256"
        state: present
      become: true
      notify:
        - hardening - restart sshd

    - name: Ensure SSH banner is configured
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?Banner'
        line: "Banner /etc/issue.net"
        state: present
      become: true
      notify:
        - hardening - restart sshd

    - name: Deploy authorized access warning banner
      ansible.builtin.copy:
        content: |
          ******************************************************************
          This system is for authorized use only. All activity is monitored
          and recorded. Unauthorized access is prohibited and will be
          prosecuted to the fullest extent of the law.
          ******************************************************************
        dest: /etc/issue.net
        owner: root
        group: root
        mode: "0644"
      become: true
```

## Firewall Rule Validation

Ensure the host firewall is active and configured to deny all inbound traffic except explicitly allowed services.

```yaml
- name: Validate and enforce firewall configuration
  tags:
    - security
    - firewall
  block:
    - name: Ensure firewalld is installed and running
      ansible.builtin.service:
        name: firewalld
        state: started
        enabled: true
      become: true
      when: ansible_os_family == "RedHat"

    - name: Set default zone to drop for firewalld
      ansible.posix.firewalld:
        zone: drop
        state: enabled
        permanent: true
        immediate: true
      become: true
      when: ansible_os_family == "RedHat"

    - name: Allow only approved inbound services
      ansible.posix.firewalld:
        zone: drop
        service: "{{ item }}"
        state: enabled
        permanent: true
        immediate: true
      become: true
      loop: "{{ hardening_allowed_services | default(['ssh']) }}"
      when: ansible_os_family == "RedHat"

    - name: Allow approved inbound ports
      ansible.posix.firewalld:
        zone: drop
        port: "{{ item }}"
        state: enabled
        permanent: true
        immediate: true
      become: true
      loop: "{{ hardening_allowed_ports | default([]) }}"
      when: ansible_os_family == "RedHat"

    - name: Verify no unexpected ports are open
      ansible.builtin.shell:
        cmd: >
          ss -tlnp
          | grep LISTEN
          | awk '{print $4}'
          | sed 's/.*://'
          | sort -un
      register: open_ports
      changed_when: false
      become: true

    - name: Report open ports for review
      ansible.builtin.debug:
        msg: >-
          Open listening ports on {{ inventory_hostname }}:
          {{ open_ports.stdout_lines | join(', ') }}.
          Approved ports: {{ hardening_approved_listen_ports | join(', ') }}.
```

## File Permission Auditing

Audit critical system files for correct ownership and permissions. World-writable files and files with setuid/setgid bits are flagged for review.

```yaml
- name: Audit file permissions on critical system paths
  tags:
    - security
    - audit
    - permissions
  block:
    - name: Find world-writable files outside of /tmp and /var/tmp
      ansible.builtin.shell:
        cmd: >
          find / -xdev -type f -perm -0002
          -not -path "/tmp/*"
          -not -path "/var/tmp/*"
          -not -path "/proc/*"
          -not -path "/sys/*"
          2>/dev/null || true
      register: world_writable_files
      changed_when: false
      become: true

    - name: Flag world-writable files for review
      ansible.builtin.debug:
        msg: >-
          World-writable files found:
          {{ world_writable_files.stdout_lines | default(['none']) | join(', ') }}
      when: world_writable_files.stdout_lines | length > 0

    - name: Find files with setuid bit set
      ansible.builtin.shell:
        cmd: >
          find / -xdev -type f -perm -4000
          -not -path "/proc/*"
          -not -path "/sys/*"
          2>/dev/null || true
      register: setuid_files
      changed_when: false
      become: true

    - name: Verify setuid files against approved list
      ansible.builtin.assert:
        that:
          - item in hardening_approved_setuid_files
        fail_msg: "Unapproved setuid file found: {{ item }}"
        success_msg: "{{ item }} is in the approved setuid list"
      loop: "{{ setuid_files.stdout_lines }}"
      when: setuid_files.stdout_lines | length > 0
      ignore_errors: true

    - name: Verify critical file permissions
      ansible.builtin.file:
        path: "{{ item.path }}"
        owner: "{{ item.owner }}"
        group: "{{ item.group }}"
        mode: "{{ item.mode }}"
      become: true
      loop:
        - { path: "/etc/passwd", owner: "root", group: "root", mode: "0644" }
        - { path: "/etc/shadow", owner: "root", group: "root", mode: "0000" }
        - { path: "/etc/group", owner: "root", group: "root", mode: "0644" }
        - { path: "/etc/gshadow", owner: "root", group: "root", mode: "0000" }
        - { path: "/etc/ssh/sshd_config", owner: "root", group: "root", mode: "0600" }
        - { path: "/etc/crontab", owner: "root", group: "root", mode: "0600" }
      loop_control:
        label: "{{ item.path }}"
```

## Service Minimization

Disable unnecessary services to reduce the attack surface. Only services required for the server's function should be running.

```yaml
- name: Minimize running services per CIS and organizational policy
  tags:
    - security
    - services
  block:
    - name: Gather list of all enabled services
      ansible.builtin.service_facts:

    - name: Disable services not in the approved list
      ansible.builtin.service:
        name: "{{ item }}"
        state: stopped
        enabled: false
      become: true
      loop: "{{ hardening_services_to_disable }}"
      when: "item + '.service' in ansible_facts.services and ansible_facts.services[item + '.service'].status == 'enabled'"

    - name: Remove unnecessary packages that provide disabled services
      ansible.builtin.yum:
        name: "{{ hardening_packages_to_remove }}"
        state: absent
      become: true
      when: ansible_os_family == "RedHat"
```

Default list of services to disable (customize per server role):

```yaml
hardening_services_to_disable:
  - avahi-daemon
  - cups
  - rpcbind
  - nfs-server
  - nfs-lock
  - nfs-idmap
  - nfs-mountd
  - postfix
  - telnet
  - rsh
  - rlogin
  - vsftpd
  - tftp
  - xinetd
  - bluetooth

hardening_packages_to_remove:
  - avahi
  - cups
  - rpcbind
  - nfs-utils
  - telnet-server
  - rsh-server
  - vsftpd
  - tftp-server
  - xinetd
```

## Handlers

Include these handlers in the role that applies hardening tasks:

```yaml
- name: hardening - restart sshd
  ansible.builtin.service:
    name: sshd
    state: restarted
  become: true

- name: hardening - reload firewalld
  ansible.builtin.service:
    name: firewalld
    state: reloaded
  become: true
```
