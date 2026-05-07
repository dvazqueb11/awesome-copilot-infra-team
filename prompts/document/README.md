# Prompts for Documenting Infrastructure Code

This collection contains prompts for generating operational documentation, architecture records, onboarding guides, and troubleshooting references from existing infrastructure code. Copy the prompt text from the fenced block and paste it into a Copilot session.

---

## Generate a runbook from a playbook

Use this when you need operational documentation that an on-call engineer can follow to execute a deployment or maintenance procedure.

```
Read this Ansible playbook and generate a complete operational runbook in Markdown format. The runbook must include every section listed below, with no sections omitted even if information must be inferred:

1. Title and metadata: playbook path, owner (HSBC Infrastructure Team), date, approval requirements
2. Purpose: two to three sentences explaining what the playbook does and which business service depends on it
3. Prerequisites: checklist of access, tools, vault passwords, inventory files, and approvals needed before execution
4. Variables table: every variable the playbook uses, whether it is required or has a default, and a description
5. Pre-flight checks: commands to verify connectivity and current state before running
6. Execution steps: the exact ansible-playbook command to run, including inventory, vault, and any extra-vars options
7. Tag-based partial execution: which tags are available and what subset of tasks each tag runs
8. Expected output: what a successful run looks like, including expected play recap numbers
9. Rollback procedure: step-by-step instructions to revert changes if the playbook fails or produces unexpected results
10. Troubleshooting: three to five common failure scenarios specific to this playbook, with symptoms, causes, and resolution steps
11. Escalation path: who to contact and what information to provide if troubleshooting does not resolve the issue

Write for an operator who did not author the playbook. Include exact commands, not vague instructions.
```

---

## Create an architecture decision record

Use this when the team makes a significant technical decision that should be documented for future reference.

```
Create an Architecture Decision Record (ADR) for the following decision:

Decision: [describe the decision, e.g., "Migrate from static inventory to dynamic inventory using ServiceNow CMDB"]

Use the following ADR format:

# ADR-[number]: [Title]

## Status
[Proposed / Accepted / Deprecated / Superseded]

## Context
Describe the technical and business context that led to this decision. What problem are we solving? What constraints exist? Include relevant details about the current state of the infrastructure, team capabilities, and organizational requirements. Be specific - reference actual systems, tools, and processes.

## Decision
State the decision clearly in one or two sentences. Then explain the details of the chosen approach, including specific technologies, patterns, and implementation details.

## Consequences

### Positive
- List each benefit of this decision with a concrete explanation

### Negative
- List each drawback or risk with a concrete explanation

### Neutral
- List changes that are neither positive nor negative but need to be accounted for

## Alternatives Considered

### [Alternative 1 name]
Describe what this alternative would look like and why it was not chosen.

### [Alternative 2 name]
Describe what this alternative would look like and why it was not chosen.

## Implementation Notes
Describe the high-level implementation plan: what changes are needed, in what order, and any migration steps required.

Do not use vague language. Every point should reference specific technologies, files, or processes.
```

---

## Generate an onboarding guide for new team members

Use this to create a guide that helps new engineers get productive with the team's infrastructure automation codebase.

```
Generate an onboarding guide for a new engineer joining the HSBC Infrastructure team. The guide should cover:

1. Repository structure: explain every top-level directory and its purpose, with examples of when an engineer would work in each area

2. Development environment setup:
   - Required software and versions (Python, Ansible, molecule, ansible-lint, yamllint)
   - How to install dependencies
   - How to configure the vault password for local development
   - How to set up SSH keys for accessing test environments
   - IDE recommendations and useful extensions

3. Key concepts: explain the team's approach to:
   - Inventory organization (environments, groups, host_vars, group_vars)
   - Role structure and naming conventions
   - Variable precedence and naming standards
   - Secret management with Ansible vault
   - Testing with molecule

4. Common workflows with step-by-step instructions:
   - How to create a new role
   - How to modify an existing playbook
   - How to run playbooks against staging
   - How to run molecule tests locally
   - How to add or rotate vault-encrypted secrets

5. Code review checklist: what reviewers look for when approving playbook changes

6. Troubleshooting: how to debug common development environment issues (vault errors, SSH failures, collection version mismatches)

7. Contacts and resources: where to find help, internal documentation links, and team communication channels

Write the guide assuming the reader is experienced with Linux systems but new to this specific codebase and team practices.
```

---

## Document the inventory structure

Use this to create documentation that explains how the inventory is organized and how to add new hosts.

```
Analyze the inventory directory structure in this project and generate comprehensive documentation that covers:

1. Directory layout: describe each directory and file under inventory/, explaining the purpose of each level of organization

2. Group hierarchy: create a text diagram showing the group relationships (parent groups, child groups, and which hosts belong to each)

3. Variable precedence: explain which variable files apply at each level and the order of precedence. Include specific examples showing how a variable defined in host_vars overrides one in group_vars.

4. How to add a new host: step-by-step instructions for adding a host to each environment, including:
   - Which inventory file to edit
   - What host-level variables to define
   - How to verify the host is correctly added (ansible-inventory --list)
   - How to test connectivity to the new host

5. How to add a new host group: instructions for creating a new logical group, adding it to the group hierarchy, and defining group-level variables

6. How to add a new environment: instructions for creating a complete new environment directory with all required variable files

7. Dynamic inventory sources: if any dynamic inventory scripts or plugins are used, explain how they work, what they connect to, and how their output merges with static inventory

Include examples using anonymized but realistic hostnames and IP addresses.
```

---

## Create a troubleshooting guide for a specific service

Use this when a team manages a service and needs a reference for common operational issues.

```
Create a troubleshooting guide for [service name] running on [platform]. The guide should cover these sections:

1. Service overview: one paragraph describing what the service does and its dependencies

2. Health check procedures:
   - How to check if the service is running (systemctl, process check)
   - How to verify the service is healthy (health endpoint, log inspection)
   - How to check connectivity to dependencies (database, upstream APIs, message queues)

3. Common issues and resolutions: for each issue, provide:
   - Symptom: what the operator observes (error message, log entry, monitoring alert)
   - Diagnostic commands: exact commands to run to confirm the root cause
   - Root cause: what is happening and why
   - Resolution: step-by-step fix, including commands
   - Prevention: how to prevent this issue from recurring

   Cover at least these scenarios:
   - Service fails to start after deployment
   - Service starts but health check fails
   - Service is running but responding slowly
   - Service cannot connect to the database
   - Service is consuming excessive memory or CPU
   - Configuration file errors after a template change
   - TLS certificate issues

4. Log locations and useful log queries:
   - Where logs are stored
   - How to filter logs for errors and warnings
   - How to correlate logs across multiple services

5. Escalation criteria: when to escalate rather than continue troubleshooting

Write for an on-call engineer who may not be deeply familiar with the service internals.
```

---

## Generate a change management summary from a pull request

Use this to produce a summary suitable for a change advisory board or change management ticket.

```
Analyze the changes in this pull request and generate a change management summary with these sections:

1. Change description: a non-technical summary of what is being changed, written for a change advisory board audience that includes non-engineers

2. Affected systems: list every host group, service, and infrastructure component that will be affected by this change

3. Risk assessment:
   - Impact: what happens if this change fails (service degradation, outage, data impact)
   - Likelihood: how likely is failure, based on the complexity and scope of the change
   - Risk level: Low / Medium / High with justification

4. Implementation plan:
   - Pre-change steps: what must be verified before applying the change
   - Change steps: the exact commands or procedures to apply the change
   - Verification steps: how to confirm the change was applied correctly
   - Duration estimate: how long the change window needs to be

5. Rollback plan: step-by-step procedure to revert the change if issues occur

6. Testing evidence: what testing was performed (molecule tests, staging deployment, check mode dry run) and the results

7. Communication plan: who needs to be notified before, during, and after the change

Keep the language clear and avoid jargon that a non-technical reviewer would not understand.
```

---

## Document role dependencies and interaction patterns

Use this when the project has multiple roles that interact and new engineers need to understand the relationships.

```
Analyze all roles in this project and generate documentation that maps their dependencies and interaction patterns:

1. Role inventory: list every role with a one-sentence description of its purpose

2. Dependency graph: create a text-based diagram showing which roles depend on which other roles (from meta/main.yml dependencies and implicit ordering in playbooks)

3. Variable flow: for each pair of roles that interact, document which variables are passed between them, where those variables are defined, and what happens if a variable is missing or has an unexpected value

4. Execution order: document the typical order in which roles are applied and explain why the ordering matters (e.g., "the firewall role must run before the application role because the application role assumes the required ports are already opened")

5. Shared resources: identify any resources (files, services, users, groups) that multiple roles manage or depend on, and document the expected state each role assumes

6. Common modification patterns: for each role, describe the most common changes engineers make and which other roles might be affected by those changes

Present this as a reference document that engineers consult before modifying roles to understand the blast radius of their changes.
```
