---
name: playbook-patterns
description: Common Ansible playbook patterns used by the infrastructure team, including rolling deployments, blue-green deployments, configuration drift detection, service health verification, and secrets rotation. Reference this skill when writing or reviewing deployment playbooks.
---

# Playbook Patterns

This skill contains standard patterns for common infrastructure automation scenarios. Each pattern includes a description, when to use it, and a complete working example. These patterns reflect the team's conventions and should be adapted to specific use cases rather than copied verbatim.

## Rolling Deployment Pattern

Use this pattern when deploying updates to a group of servers where the service must remain available throughout the deployment. The playbook processes hosts in batches, verifying health after each batch before continuing.

```yaml
---
- name: Rolling deployment of application service
  hosts: app_servers
  become: true
  serial: "25%"
  max_fail_percentage: 0
  any_errors_fatal: false
  gather_facts: true

  pre_tasks:
    - name: Remove host from load balancer pool
      ansible.builtin.uri:
        url: "https://{{ lb_api_endpoint }}/api/v1/pools/{{ lb_pool_id }}/members/{{ ansible_host }}"
        method: DELETE
        headers:
          Authorization: "Bearer {{ vault_lb_api_token }}"
        status_code: [200, 204]
        validate_certs: true
      delegate_to: localhost
      tags:
        - deploy
        - lb

    - name: Wait for active connections to drain
      ansible.builtin.wait_for:
        timeout: "{{ rolling_deploy_drain_timeout | default(30) }}"
      tags:
        - deploy

  tasks:
    - name: Deploy application update
      tags:
        - deploy
        - install
      block:
        - name: Stop application service before deployment
          ansible.builtin.service:
            name: "{{ app_service_name }}"
            state: stopped

        - name: Deploy application package
          ansible.builtin.yum:
            name: "{{ app_package_name }}-{{ app_target_version }}"
            state: present
          register: package_result

        - name: Deploy updated configuration
          ansible.builtin.template:
            src: app_config.yml.j2
            dest: "/etc/{{ app_service_name }}/config.yml"
            owner: "{{ app_service_user }}"
            group: "{{ app_service_group }}"
            mode: "0640"
          notify:
            - app - restart service

        - name: Start application service after deployment
          ansible.builtin.service:
            name: "{{ app_service_name }}"
            state: started
            enabled: true

      rescue:
        - name: Log deployment failure details
          ansible.builtin.debug:
            msg: >-
              Deployment failed on {{ inventory_hostname }}.
              Package result: {{ package_result | default('not attempted') }}.
              Initiating rollback.

        - name: Rollback to previous package version
          ansible.builtin.yum:
            name: "{{ app_package_name }}-{{ app_previous_version }}"
            state: present
          when: package_result is defined and package_result is changed

        - name: Ensure service is running after rollback
          ansible.builtin.service:
            name: "{{ app_service_name }}"
            state: started

        - name: Fail the host to prevent further processing
          ansible.builtin.fail:
            msg: "Deployment failed and was rolled back on {{ inventory_hostname }}"

  post_tasks:
    - name: Verify application health before returning to pool
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}:{{ app_health_port }}/health"
        method: GET
        status_code: 200
      register: health_result
      retries: 10
      delay: 5
      until: health_result.status == 200
      tags:
        - deploy
        - verify

    - name: Add host back to load balancer pool
      ansible.builtin.uri:
        url: "https://{{ lb_api_endpoint }}/api/v1/pools/{{ lb_pool_id }}/members"
        method: POST
        headers:
          Authorization: "Bearer {{ vault_lb_api_token }}"
        body_format: json
        body:
          address: "{{ ansible_host }}"
          port: "{{ app_listen_port }}"
        status_code: [200, 201]
        validate_certs: true
      delegate_to: localhost
      tags:
        - deploy
        - lb

  handlers:
    - name: app - restart service
      ansible.builtin.service:
        name: "{{ app_service_name }}"
        state: restarted
```

Key elements of this pattern:
- `serial: "25%"` processes one quarter of hosts at a time
- Pre-tasks remove the host from the load balancer and drain connections
- A `block/rescue` construct handles deployment failure with automatic rollback
- Post-tasks verify health before returning the host to the pool
- `max_fail_percentage: 0` halts the entire deployment if any batch fails

## Blue-Green Deployment Pattern

Use this pattern when you maintain two identical environments (blue and green) and switch traffic between them during deployments. This approach provides instant rollback by redirecting traffic back to the previous environment.

```yaml
---
- name: Blue-green deployment - deploy to inactive environment
  hosts: "{{ deploy_target_color }}_servers"
  become: true
  any_errors_fatal: true
  gather_facts: true

  vars:
    deploy_active_color: "{{ lookup('file', '/opt/deploy/active_color.txt') | trim }}"
    deploy_target_color: "{{ 'green' if deploy_active_color == 'blue' else 'blue' }}"

  tasks:
    - name: Deploy application to inactive environment
      tags:
        - deploy
      block:
        - name: Deploy application package to {{ deploy_target_color }} environment
          ansible.builtin.yum:
            name: "{{ app_package_name }}-{{ app_target_version }}"
            state: present

        - name: Deploy configuration for {{ deploy_target_color }} environment
          ansible.builtin.template:
            src: app_config.yml.j2
            dest: "/etc/{{ app_service_name }}/config.yml"
            owner: "{{ app_service_user }}"
            group: "{{ app_service_group }}"
            mode: "0640"

        - name: Restart service in {{ deploy_target_color }} environment
          ansible.builtin.service:
            name: "{{ app_service_name }}"
            state: restarted
            enabled: true

        - name: Verify service health in {{ deploy_target_color }} environment
          ansible.builtin.uri:
            url: "http://{{ ansible_host }}:{{ app_health_port }}/health"
            method: GET
            status_code: 200
          register: bg_health_check
          retries: 10
          delay: 5
          until: bg_health_check.status == 200

      rescue:
        - name: Deployment to {{ deploy_target_color }} failed - environment unchanged
          ansible.builtin.fail:
            msg: >-
              Deployment to {{ deploy_target_color }} environment failed.
              Active environment ({{ deploy_active_color }}) is unaffected.
              Investigate and retry.

- name: Blue-green deployment - switch traffic to new environment
  hosts: load_balancers
  become: true
  any_errors_fatal: true

  tasks:
    - name: Switch load balancer to {{ deploy_target_color }} environment
      tags:
        - deploy
        - switch
      block:
        - name: Update load balancer backend pool to {{ deploy_target_color }}
          ansible.builtin.template:
            src: lb_backend.conf.j2
            dest: /etc/nginx/conf.d/backend.conf
            owner: root
            group: root
            mode: "0644"
          vars:
            active_backend_group: "{{ deploy_target_color }}_servers"

        - name: Reload load balancer configuration
          ansible.builtin.service:
            name: nginx
            state: reloaded

        - name: Verify traffic is routing to {{ deploy_target_color }} environment
          ansible.builtin.uri:
            url: "https://{{ service_fqdn }}/health"
            method: GET
            status_code: 200
            return_content: true
          register: switch_verify
          retries: 5
          delay: 3
          until: switch_verify.status == 200
          delegate_to: localhost

        - name: Record new active environment color
          ansible.builtin.copy:
            content: "{{ deploy_target_color }}"
            dest: /opt/deploy/active_color.txt
            mode: "0644"
          delegate_to: localhost

      rescue:
        - name: Switch failed - revert to {{ deploy_active_color }} environment
          ansible.builtin.template:
            src: lb_backend.conf.j2
            dest: /etc/nginx/conf.d/backend.conf
            owner: root
            group: root
            mode: "0644"
          vars:
            active_backend_group: "{{ deploy_active_color }}_servers"

        - name: Reload load balancer after reverting backend
          ansible.builtin.service:
            name: nginx
            state: reloaded

        - name: Fail after reverting to previous environment
          ansible.builtin.fail:
            msg: "Traffic switch failed. Reverted to {{ deploy_active_color }} environment."
```

## Configuration Drift Detection

Use this pattern to detect whether the actual state of servers has drifted from the desired state defined in your playbooks. Run this on a schedule or before deployments to identify manual changes.

```yaml
---
- name: Detect configuration drift on managed servers
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Check for configuration drift in managed files
      tags:
        - verify
        - drift
      block:
        - name: Run playbook in check mode and capture changes
          ansible.builtin.shell:
            cmd: >
              ansible-playbook site.yml
              --check --diff
              -i {{ inventory_file }}
              --limit {{ inventory_hostname }}
              2>&1
          register: drift_check_output
          delegate_to: localhost
          changed_when: false
          tags:
            - drift

        - name: Parse drift results for changed tasks
          ansible.builtin.set_fact:
            drift_detected: "{{ 'changed=' in drift_check_output.stdout and 'changed=0' not in drift_check_output.stdout }}"
            drift_details: "{{ drift_check_output.stdout_lines | select('match', '.*changed:.*') | list }}"
          tags:
            - drift

        - name: Report drift status for {{ inventory_hostname }}
          ansible.builtin.debug:
            msg: >-
              Drift detection for {{ inventory_hostname }}:
              Drift detected: {{ drift_detected }}.
              Details: {{ drift_details | join(', ') }}
          tags:
            - drift

    - name: Verify critical file checksums against known-good values
      tags:
        - verify
        - drift
        - checksums
      block:
        - name: Calculate checksums of critical configuration files
          ansible.builtin.stat:
            path: "{{ item.path }}"
            checksum_algorithm: sha256
          register: file_checksums
          loop: "{{ drift_monitored_files }}"
          loop_control:
            label: "{{ item.path }}"

        - name: Compare checksums against expected values
          ansible.builtin.assert:
            that:
              - item.stat.checksum == drift_expected_checksums[item.item.path]
            fail_msg: >-
              DRIFT DETECTED: {{ item.item.path }} has been modified.
              Expected checksum: {{ drift_expected_checksums[item.item.path] }}.
              Actual checksum: {{ item.stat.checksum }}.
            success_msg: "{{ item.item.path }} matches expected checksum"
          loop: "{{ file_checksums.results }}"
          loop_control:
            label: "{{ item.item.path }}"
          when:
            - item.stat.exists
            - item.item.path in drift_expected_checksums
          ignore_errors: true
          register: checksum_results

        - name: Compile drift report
          ansible.builtin.set_fact:
            drift_report:
              hostname: "{{ inventory_hostname }}"
              timestamp: "{{ ansible_date_time.iso8601 }}"
              files_checked: "{{ file_checksums.results | length }}"
              drift_found: "{{ checksum_results.results | selectattr('failed', 'defined') | selectattr('failed') | list | length }}"
              details: "{{ checksum_results.results | selectattr('failed', 'defined') | selectattr('failed') | map(attribute='item.item.path') | list }}"
```

## Service Health Verification

Use this pattern after any deployment or configuration change to verify services are running correctly. This pattern checks multiple health indicators and produces a structured report.

```yaml
---
- name: Comprehensive service health verification
  hosts: app_servers
  become: true
  gather_facts: true

  tasks:
    - name: Verify service process and connectivity
      tags:
        - verify
        - health
      block:
        - name: Gather service facts from systemd
          ansible.builtin.service_facts:

        - name: Assert the application service is running
          ansible.builtin.assert:
            that:
              - "'{{ app_service_name }}.service' in ansible_facts.services"
              - "ansible_facts.services['{{ app_service_name }}.service'].state == 'running'"
            fail_msg: "Service {{ app_service_name }} is not in running state"
            success_msg: "Service {{ app_service_name }} is running"

        - name: Check application HTTP health endpoint
          ansible.builtin.uri:
            url: "http://{{ ansible_host }}:{{ app_health_port }}/health"
            method: GET
            status_code: 200
            return_content: true
            timeout: 10
          register: http_health

        - name: Verify application readiness endpoint
          ansible.builtin.uri:
            url: "http://{{ ansible_host }}:{{ app_health_port }}/ready"
            method: GET
            status_code: 200
            return_content: true
            timeout: 10
          register: readiness_check

        - name: Verify application can reach database
          ansible.builtin.uri:
            url: "http://{{ ansible_host }}:{{ app_health_port }}/health/db"
            method: GET
            status_code: 200
            timeout: 10
          register: db_health

        - name: Check application log for error patterns in recent entries
          ansible.builtin.shell:
            cmd: >
              journalctl -u {{ app_service_name }}
              --since "5 minutes ago"
              --no-pager
              --output cat
              | grep -ci "ERROR\|FATAL\|CRITICAL" || true
          register: log_errors
          changed_when: false

        - name: Verify disk space is within acceptable thresholds
          ansible.builtin.shell:
            cmd: >
              df -h {{ app_data_directory }}
              | tail -1
              | awk '{print $5}'
              | sed 's/%//'
          register: disk_usage
          changed_when: false

        - name: Assert disk usage is below threshold
          ansible.builtin.assert:
            that:
              - disk_usage.stdout | int < 85
            fail_msg: "Disk usage at {{ disk_usage.stdout }}% exceeds 85% threshold"
            success_msg: "Disk usage at {{ disk_usage.stdout }}% is within acceptable range"

        - name: Compile health verification report
          ansible.builtin.debug:
            msg:
              host: "{{ inventory_hostname }}"
              service_state: "{{ ansible_facts.services[app_service_name + '.service'].state }}"
              http_health: "{{ http_health.status }}"
              readiness: "{{ readiness_check.status }}"
              database_connectivity: "{{ db_health.status }}"
              recent_errors: "{{ log_errors.stdout | int }}"
              disk_usage_percent: "{{ disk_usage.stdout }}"
              overall: "{{ 'HEALTHY' if (http_health.status == 200 and log_errors.stdout | int == 0) else 'DEGRADED' }}"
```

## Secrets Rotation Pattern

Use this pattern when rotating credentials such as database passwords, API tokens, or TLS certificates. The pattern ensures zero-downtime rotation by updating the service configuration and verifying health before finalizing the change.

```yaml
---
- name: Rotate application database credentials
  hosts: app_servers
  become: true
  serial: 1
  any_errors_fatal: true
  gather_facts: true

  vars_files:
    - vars/vault_db_credentials.yml

  tasks:
    - name: Rotate database credentials with zero-downtime
      tags:
        - security
        - secrets
      block:
        - name: Verify current credentials are working before rotation
          ansible.builtin.uri:
            url: "http://{{ ansible_host }}:{{ app_health_port }}/health/db"
            method: GET
            status_code: 200
            timeout: 10
          register: pre_rotation_health

        - name: Deploy updated configuration with new database credentials
          ansible.builtin.template:
            src: db_connection.yml.j2
            dest: "/etc/{{ app_service_name }}/db_connection.yml"
            owner: "{{ app_service_user }}"
            group: "{{ app_service_group }}"
            mode: "0600"
          vars:
            db_password: "{{ vault_db_password_new }}"

        - name: Gracefully reload the application to pick up new credentials
          ansible.builtin.service:
            name: "{{ app_service_name }}"
            state: reloaded

        - name: Wait for application to stabilize after credential change
          ansible.builtin.wait_for:
            timeout: 10

        - name: Verify database connectivity with new credentials
          ansible.builtin.uri:
            url: "http://{{ ansible_host }}:{{ app_health_port }}/health/db"
            method: GET
            status_code: 200
            timeout: 10
          register: post_rotation_health
          retries: 5
          delay: 3
          until: post_rotation_health.status == 200

      rescue:
        - name: Rotation failed - restore previous credentials
          ansible.builtin.template:
            src: db_connection.yml.j2
            dest: "/etc/{{ app_service_name }}/db_connection.yml"
            owner: "{{ app_service_user }}"
            group: "{{ app_service_group }}"
            mode: "0600"
          vars:
            db_password: "{{ vault_db_password_current }}"

        - name: Reload application with restored credentials
          ansible.builtin.service:
            name: "{{ app_service_name }}"
            state: reloaded

        - name: Verify service recovered after credential rollback
          ansible.builtin.uri:
            url: "http://{{ ansible_host }}:{{ app_health_port }}/health/db"
            method: GET
            status_code: 200
          register: rollback_health
          retries: 5
          delay: 3
          until: rollback_health.status == 200

        - name: Fail with rotation error details
          ansible.builtin.fail:
            msg: >-
              Credential rotation failed on {{ inventory_hostname }}.
              Previous credentials have been restored and verified.
              Investigate before retrying.

      always:
        - name: Record rotation attempt in audit log
          ansible.builtin.lineinfile:
            path: /var/log/credential_rotation.log
            line: >-
              {{ ansible_date_time.iso8601 }} |
              host={{ inventory_hostname }} |
              credential=db_password |
              result={{ 'success' if post_rotation_health is defined and post_rotation_health.status == 200 else 'failed' }}
            create: true
            mode: "0600"
            owner: root
            group: root
```
