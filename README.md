# awesome-copilot-infra

A curated collection of GitHub Copilot agents, skills, prompts, and coding instructions built for the Infrastructure team. This repository follows the GitHub awesome-copilot pattern, tailored specifically for teams working with Python and Ansible to automate cloud infrastructure provisioning, configuration management, and operational workflows.

Use this repository as a Copilot CLI plugin to get purpose-built AI assistance that understands your team's standards, patterns, and toolchain.

## What is included

| Directory | Contents |
|-----------|----------|
| `agents/` | Custom Copilot agents for playbook review, runbook generation, and role scaffolding |
| `skills/` | Reusable knowledge packs covering deployment patterns, security hardening, and troubleshooting |
| `prompts/` | Copy-pasteable prompt templates organized by workflow - creating, improving, and documenting infrastructure code |
| `instructions/` | Team coding standards for Ansible and Python that agents reference during reviews and generation |
| `.github/copilot-instructions.md` | Baseline instructions applied to every Copilot interaction in projects that adopt this plugin |

## Quick start

### Install as a Copilot CLI plugin

Clone this repository and install it as a local plugin:

```bash
git clone <repo-url> awesome-copilot-infra
cd awesome-copilot-infra
copilot plugin install ./
```

Verify the installation:

```bash
copilot plugin list
```

In an interactive Copilot session, confirm agents and skills are loaded:

```
/agent
/skills list
```

### Use individual components

If you prefer not to install the full plugin, copy individual components into your project:

- Copy an agent file into your project's `.github/agents/` directory
- Copy a skill directory into your project's `.github/skills/` directory
- Copy `.github/copilot-instructions.md` into your project's `.github/` directory

## Directory structure

```
awesome-copilot-infra/
  plugin.json                          # Plugin manifest for Copilot CLI
  .github/
    copilot-instructions.md            # Team-wide Copilot behavior instructions
  agents/
    ansible-reviewer.agent.md          # Reviews playbooks for standards compliance
    runbook-generator.agent.md         # Generates runbooks from playbook files
    infra-scaffolder.agent.md          # Scaffolds new roles and playbooks
  skills/
    playbook-patterns/
      SKILL.md                         # Common deployment and operations patterns
    hardening-checklist/
      SKILL.md                         # CIS and security hardening reference
    troubleshooting/
      SKILL.md                         # Debugging failed deployments
  prompts/
    create/
      README.md                        # Prompts for creating new infrastructure code
    improve/
      README.md                        # Prompts for improving existing code
    document/
      README.md                        # Prompts for generating documentation
  instructions/
    ansible-standards.md               # Ansible coding standards reference
    python-standards.md                # Python coding standards reference
```

## How agents work

Each agent is a Markdown file with YAML frontmatter that defines the agent's name, description, and available tools. When you install this plugin, Copilot automatically routes relevant requests to the appropriate agent based on the description field.

For example, asking Copilot to "review this playbook for security issues" will route to the `ansible-reviewer` agent, which understands your team's Ansible standards and security requirements.

## How skills work

Skills are knowledge packs that Copilot can reference during any conversation. They contain patterns, checklists, and reference material specific to your team's work. Unlike agents, skills do not define behavior - they provide context that any agent or general Copilot interaction can draw from.

## How prompts work

The `prompts/` directory contains ready-to-use prompt templates organized by workflow. These are not consumed by Copilot automatically - they are reference material for team members to copy and paste into Copilot sessions or adapt for their specific tasks.

## Contributing

### Adding a new agent

1. Create a new file in `agents/` with the `.agent.md` extension.
2. Include YAML frontmatter with `name`, `description`, and `tools` fields.
3. Write clear instructions in the Markdown body explaining the agent's behavior.
4. Test locally by reinstalling the plugin: `copilot plugin install ./`
5. Open a pull request with a description of the agent's purpose and example usage.

### Adding a new skill

1. Create a new subdirectory under `skills/` named after the skill.
2. Add a `SKILL.md` file with YAML frontmatter containing `name` and `description`.
3. Include actionable reference material, code snippets, and checklists in the body.
4. Open a pull request.

### Adding new prompts

1. Determine which category the prompt belongs to: `create`, `improve`, or `document`.
2. Add the prompt to the appropriate `README.md` file in `prompts/`.
3. Include the full prompt text as a fenced code block so it is easy to copy.
4. Add a brief description above the block explaining when to use the prompt.

### Standards for contributions

- All content must follow the team's Ansible and Python coding standards documented in `instructions/`.
- No hardcoded credentials, hostnames, or environment-specific values in examples.
- Use realistic but anonymized examples. Replace real server names with patterns like `app-server-01.example.internal`.
- Keep agent prompts under 30,000 characters.
- Test agents and skills with representative workloads before submitting.

## Updating the plugin

After making changes, team members who have installed the plugin should reinstall to pick up updates:

```bash
cd awesome-copilot-infra
git pull
copilot plugin install ./
```

## License

Internal use only. This repository and its contents are proprietary to HSBC Infrastructure and must not be distributed outside the organization.
