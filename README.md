# Deploy the open-source AI Operating System on your own infrastructure

Kortix is the open-source AI Operating System. This repo holds the deployment half of it: an annotated `kortix.yaml`, the self-host commands, and the review path, in files you can read before you run anything.

Self-hosting it looks like an infrastructure project. It is a git project first. The manifest and the branch are the parts you keep when a server dies, and they decide what the system is allowed to do. Compute you can rent again. That is the argument this repo makes, with the files to check it.

## Start in three commands

```bash
curl -fsSL https://kortix.com/install | bash
kortix self-host start
kortix hosts use selfhost
```

The installer puts the CLI on macOS or Linux. On Windows, run it in WSL. `kortix self-host start` pulls the stack's images and brings up one Docker Compose project; a first run asks the setup questions and then starts. `kortix hosts use selfhost` points the CLI at your instance, so `kortix sessions`, `kortix cr` and `kortix agents` act on your server instead of Kortix Cloud.

Run as written, that brings up a local instance and expects Docker and the Compose plugin on the host. A production instance wants a domain and a certificate first, because a local stack binds every port to `127.0.0.1` and a sandbox cannot reach the API to start a session. The bootstrap in [docs/self-host.md](docs/self-host.md) installs Docker, sets the domain and covers the first sign-in.

## What is in git, and what is not

| In `kortix.yaml`, in your repo | In the running instance |
| --- | --- |
| The default agent and each agent's grants | Secret values and connector accounts |
| Triggers, their schedule and their prompt | Each trigger's last fire, status and error |
| Connector declarations and secret names | Members, groups and roles |
| Sandbox templates and harness settings | Sessions, branches and change requests |

`kortix init` scaffolds the left column into a new project. The right column is state the platform records while it runs. Both matter, and only one of them is a file you can diff. A grant you leave out resolves to `none`, so a connector that no agent lists is a connector no agent can call.

[docs/kortix-yaml.md](docs/kortix-yaml.md) walks through a full manifest for a small company, with every line explained.

## The review path is the control

A session clones the project, cuts a branch named after the session, and runs one agent. The agent commits and pushes that branch. Then it opens a change request, and a person reads the diff and merges it.

```bash
kortix cr ls
kortix cr diff 1
kortix cr merge 1
```

Session work reaches `main` through a change request. Merge is default-deny for agents. To let an agent merge, an admin grants `project.cr.merge` explicitly; the example manifest does not.

Agents reach tools through connectors the project declares, and each agent's `connectors` grant decides which ones. Connector credentials are brokered server-side and never enter the machine.

## Limits, stated once

`kortix self-host start` pulls its images from Docker Hub, and every session runs on the sandbox provider you configure, which is Daytona by default or Platinum or E2B. The instance must be reachable from the internet so those sandboxes can call back to its API. This is a self-hosted install, not an air-gapped one. If you need an isolated topology, scope it with the Kortix team before you start.

## The three docs

- [docs/self-host.md](docs/self-host.md): prerequisites, the bootstrap, where the database, repos, secrets and audit record live, and how to update, back up and switch hosts.
- [docs/kortix-yaml.md](docs/kortix-yaml.md): an annotated v2 manifest with agents, skills, a cron trigger, explicit grants and a policy.
- [docs/faq.md](docs/faq.md): six questions a platform engineer asks before deploying this, each answered in a sentence or two.

## Where the commands come from

Every command and key in these files comes from Kortix's own documentation: the self-hosting page at `kortix.com/docs/host`, the `kortix.yaml` reference at `kortix.com/docs/project/manifest`, the CLI reference at `kortix.com/docs/cli`, and the two blog posts on getting started and on the platform's layers. [Kortix on GitHub](https://github.com/kortix-ai/suna) carries the code. The ownership argument, in shorter form, is at [opensourceaioperatingsystem.com](https://opensourceaioperatingsystem.com/).

Run it on Kortix Cloud, in your VPC, or on your own on-prem network. Self-host is free. [Try Kortix](https://kortix.com).
