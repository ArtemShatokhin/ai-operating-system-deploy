# Six questions before you self-host

Kortix is the open-source AI Operating System. These are the six things a platform engineer wants settled before the stack goes on a company network.

## What do I need before I run the bootstrap?

A Linux host that answers on a public address, Docker with the Compose plugin, DNS for the app and API names, inbound 80 and 443, a sandbox provider key, and a model provider key. The bootstrap can install Docker for you.

## What do I own when I self-host?

The project repo and the instance directory. Agent prompts, skills, memory and connector configuration are versioned files you can clone; the Postgres volume, uploaded files and `.env` live under `~/.config/kortix/self-host/<instance>/`. Compute belongs to the sandbox provider.

## What stays outside my infrastructure?

Sandboxes, inference, and the images. Sessions run on Daytona, Platinum or E2B under your key, model traffic goes to whichever provider each project connects, and the stack pulls from Docker Hub. Sandboxes dial back to the API, so the host needs a public route.

## How do agents reach my tools?

Through connectors declared in `kortix.yaml`, with each agent's `connectors` grant naming the ones it may call. Connector credentials are brokered server-side and never enter the machine; the sandbox holds a project-scoped Kortix token instead of your API keys.

## How does an agent's change reach `main`?

It pushes its branch and opens a change request. A person reads the diff and merges it, and an agent can merge only if an admin has granted `project.cr.merge` in its permissions.

## Can I move to managed cloud later?

Yes. Run it on Kortix Cloud, in your VPC, or on your own on-prem network. Self-host is free. `kortix hosts use cloud` points the CLI at managed hosting, and `kortix hosts use selfhost` points it back; the repo does not move.
