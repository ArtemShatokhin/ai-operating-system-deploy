# The `kortix.yaml` that runs a small company

Kortix is the open-source AI Operating System: agents, skills, memory and connector configuration live in one git repo you own. This page is about the file at its root.

`kortix.yaml` is the Kortix layer of a project repository. It declares which agents exist, what each one may reach, the triggers that start sessions with nobody present, the connectors the project offers, and the names of the secrets it expects. Secret values stay in the Secrets store, never here.

Read it as a permission file. It is the part of a self-hosted install that survives a provider change, because it is the part you keep in git.

## A manifest for Acme

Acme is a placeholder company with a support queue, a weekly operations digest and one general agent. The block below is version 2, which is what `kortix init` and the dashboard write.

```yaml
# yaml-language-server: $schema=https://kortix.com/schema/kortix.v2.schema.json
kortix_version: 2                 # picks the schema; v2 keeps agent behavior in agents/<name>.md
default_agent: chief-of-staff      # runs when a session or trigger names no agent

runtime: opencode                  # the default harness
opencode:
  config_dir: harnesses/opencode   # files only OpenCode reads

project:
  name: acme                       # for people reading the repo
  description: Acme agents, skills, memory and connector config.

env:
  required: [HELPDESK_API_KEY]     # names only; values live in the Secrets store

connectors:
  - slug: helpdesk                 # the name agents reference
    provider: http                 # composio, pipedream, mcp, openapi, postman, graphql, http or channel
    base_url: https://helpdesk.example.com
    auth:
      type: bearer

agents:
  chief-of-staff:
    file: agents/chief-of-staff.md
    connectors: all
    secrets: all
    skills: [kortix-cli, kortix-memory, weekly-report]
    kortix_permissions: all        # every project action; narrow this per agent with `kortix validate --scopes`

  support-triage:
    file: agents/support-triage.md
    connectors: [helpdesk]
    secrets: [HELPDESK_API_KEY]
    skills: [kortix-cli, triage]
    kortix_permissions: [project.cr.open]

  ops-reporter:
    file: agents/ops-reporter.md
    skills: [kortix-cli, weekly-report]
    kortix_permissions: [project.cr.open]

policy:
  default_mode: risk               # reads run; writes and destructive calls stop for a person

triggers:
  - slug: monday-ops-digest
    type: cron                     # a schedule, no person present
    agent: ops-reporter            # the trigger inherits this agent's reach
    cron: "0 0 8 * * 1"            # six fields, seconds first: 08:00 every Monday
    timezone: Europe/Berlin        # an IANA name; UTC is the default
    prompt: |
      Read last week's support and delivery notes, write the Monday
      digest, and open one change request.
```

## Line by line

The first line is an editor hint, not a Kortix key. It points a YAML language server at the version 2 schema, so the editor validates and completes every key as you type.

`kortix_version: 2` picks the schema. In version 2 the agent's behavior, meaning the prompt, model, mode and tool permission rules, lives in `agents/<name>.md`; governance lives in this file. Version 3 moves behavior inline, and no automatic migration from 2 to 3 exists. Pick one and stay on it.

`default_agent: chief-of-staff` names the agent a session or trigger runs when it names no other. It must point at a declared agent that is not disabled.

`runtime: opencode` selects the harness. OpenCode is the default. `opencode.config_dir` says where the files only OpenCode reads live, such as `opencode.jsonc` and plugins.

`project.name` and `project.description` are for people who read the repository. The project's name inside Kortix does not come from here.

`env.required` declares the secret names the project expects. The dashboard and `kortix ship` prompt for a missing value. The value itself goes to the Secrets store, and a name here that never gets a value is still only a name. A name in an agent's `secrets` grant or a trigger's `secret_env` must appear under `env`, or validation fails.

`connectors` declares what the project can call. The slug is what agents reference. This example declares one HTTP connector at a placeholder host, with a bearer credential. The manifest names the connector; the credential is stored in the platform. Connector credentials are brokered server-side and never enter the machine. A provider can be `composio`, `pipedream`, `mcp`, `openapi`, `postman`, `graphql`, `http` or `channel`.

Inside `agents`, each key is an agent name, and each block is governance only. `file` points at the markdown agent. `connectors`, `secrets` and `skills` are grant sets: `all`, `none`, or a list. `kortix_permissions` is the set of project actions the agent's service account may take.

The grants are deny-by-default. An omitted grant resolves to `none`, so `support-triage` can call the helpdesk connector and no other, and `ops-reporter` gets no connector at all. `skills` behaves slightly differently: when it is omitted, the agent file's own `permission.skill` rule decides.

The `kortix_permissions` lists above are deliberately short. `project.cr.open` is the grant that lets a restricted agent put its work up for review; `kortix validate --scopes` prints every grantable entry, so copy from that list rather than guessing a name. `project.cr.merge` appears nowhere in this project, so the merge stays with a person. Session work reaches `main` through a change request. Merge is default-deny for agents.

`policy.default_mode: risk` is the approval setting. In `risk` mode a read runs, and a write or destructive call waits for a person. Approval gates you set. Off until you set them, which is why this block exists: a project that declares no policy falls back to allowing everything.

`triggers` is the schedule. `type: cron` with a six-field expression, seconds first, so `0 0 8 * * 1` is 08:00 every Monday. `timezone` takes an IANA name and defaults to UTC. `agent` names who runs, and `prompt` is the instruction the fire renders. The trigger inherits the reach of the agent it names, so the digest job can push a branch and open a change request, and it still cannot merge.

## Validate before you push

`kortix validate` checks the manifest against the schema, lints any sandbox template Dockerfile, and warns about large files in git. It exits 0 when valid, 1 on an error and 2 when the file is missing. The same check runs in `kortix ship` and in the change request merge check.

```bash
kortix validate
kortix validate --scopes
kortix schema --version 2
```

`--scopes` prints every grantable `kortix_permissions` entry, which is the list to copy from instead of guessing a name. `kortix schema --version 2` prints the schema itself. A permission that is not on the live list fails validation and blocks the merge.

## What the manifest cannot hold

`kortix.yaml` does not declare channels. A version 2 validator rejects a `channels:` block, because channel routing is live project state. It does not hold members or roles either: people, groups and service accounts are principals in the instance, not entries in a file. Secret values, connector credentials and each trigger's last fire are also runtime state.

Everything else that shapes what an agent may do is here, in a diff someone can read. That is the point of keeping the manifest in git: a change to an agent's reach is a change request, with an author and a reviewer, not a settings screen someone clicked.

The keys and rules here are from the `kortix.yaml` reference at `kortix.com/docs/project/manifest` and the agents and permissions page at `kortix.com/docs/project/agents`; the validation behavior is from the CLI reference at `kortix.com/docs/cli`.
