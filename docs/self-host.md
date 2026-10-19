# Self-host Kortix on your own server

Kortix is the open-source AI Operating System. This page is for the operator who runs the stack: what to prepare, what to type, and where the pieces end up.

A self-hosted instance is the whole platform, not an edition. The CLI installs one Compose stack and keeps it current. Every command here is from the self-hosting and CLI reference, and every path is one the stack writes.

## What you need

- One Linux host with a public address, and Docker Engine plus the Compose plugin. The bootstrap installs both when they are missing.
- DNS. Before the first start, point `A` or `AAAA` records at the host for `kortix.example.com` and `api.kortix.example.com`, and add a wildcard `*.p.kortix.example.com` for previews. Caddy asks for the certificate once those names resolve.
- Inbound traffic allowed on 80 and 443.
- A key for Daytona, Platinum or E2B. Sessions cannot start without one.
- A model provider key, entered in the web app after sign-in. No model ships with the stack.
- A GitHub identity for managed git, connected at `/admin/git` after sign-up.

## Bring it up

On a bare host, the bootstrap installs Docker and the CLI, writes the instance files, sets the domain and brings the stack up.

```bash
curl -fsSL https://raw.githubusercontent.com/kortix-ai/suna/dev/scripts/kortix-selfhost-up.sh \
  | bash -s -- --domain kortix.example.com --email ops@example.com
```

If the script adds you to the docker group, start a new shell first. Then answer the questions and apply them:

```bash
kortix self-host configure
kortix self-host start
kortix self-host doctor
```

`configure` walks the setup questions, including reachability, admin email, sandbox provider and preview domain. It writes files and nothing else; `start` applies them. `doctor` checks the result and exits 0 when everything passes. Clear its failures before you invite anyone.

With Docker and the Compose plugin already installed, the same job runs by hand:

```bash
curl -fsSL https://kortix.com/install | bash
kortix self-host init --domain kortix.example.com --admin-email you@example.com
kortix self-host start
```

On a terminal, the first `init` asks every question. `--yes`, or a session with no terminal, takes the defaults. `start` runs `init` itself when no configuration exists.

## First sign-in, in order

1. Open `https://kortix.example.com` and sign up with an address in `KORTIX_PLATFORM_ADMIN_EMAILS`. That list makes you a platform admin.
2. Connect GitHub at `/admin/git`. Kortix creates and pushes every new project repository through this one identity; until it is set, Kortix cannot create project repositories.
3. Create a project at `/new`, then create the team account from the project menu. Your sign-up account is personal, and a personal account cannot hold SAML SSO.
4. Add a model key in the project: Customize, Models, Providers, or `kortix providers set <provider>`. Each project needs its own key.
5. Set `EMAIL_URL` before you invite people, then close sign-up with `DISABLE_SIGNUP=true` and restart. Until email is set, a password sign-up is confirmed without an email, and the first person to claim an address holds it.

```bash
kortix self-host env set EMAIL_URL=smtp://user:password@smtp.example.com:587
kortix self-host env set DISABLE_SIGNUP=true
kortix self-host start
```

## Where everything lives

`init` writes `docker-compose.yml`, `.env`, a `Caddyfile` and `updater.sh` into `~/.config/kortix/self-host/<instance>/`. Every service belongs to one Compose project, `kortix-<instance>`: frontend, `kortix-api`, `llm-gateway`, migrations, updater, Caddy, and the Supabase set with Postgres, auth, storage and the pooler.

| What | Where it lives |
| --- | --- |
| Database and stored files | `volumes/db/data` and `volumes/storage` in the instance directory |
| Secret values | The `.env` file and the encrypted Secrets store |
| Project repositories | GitHub, through the identity connected at `/admin/git` |
| Audit record | The instance's Postgres database |

The database is a volume on the server, and the repositories are not. That split is the argument of this repository in one line: the state you cannot recreate is the repo, and the server is the part you can replace.

Secrets are encrypted at rest with a key per project. On the box, `.env` holds every secret and signing key, and `API_KEY_SECRET` in it encrypts every stored secret, connector credential and token. A lost `API_KEY_SECRET` makes them unreadable, so keep the file safe and back it up with the database. A project secret whose value can no longer be decrypted is marked `Undecryptable`, and a connector call returns `credential_undecryptable`.

Every action is recorded. Reading, exporting and streaming the audit log depend on the plan. On a self-hosted instance, reading it needs an Enterprise plan; recording is not optional. The log keeps 90 days of events live and the archive keeps them up to 365 days.

## What runs where

One isolated sandbox per session. Each session has its own isolated machine and branch. That machine is created by `kortix-api` with your provider key, and it dials back to the public API URL. Model calls leave through the gateway, to the providers your projects connect. Images arrive from Docker Hub.

## Two limits

The install is not air-gapped. Images come from Docker Hub, and sandboxes have to reach the API from outside, so the host needs a public route. A local instance with no domain binds every port to `127.0.0.1`, and in that mode a session cannot start at all.

The sandboxes are remote. Daytona, Platinum or E2B runs them under your key. Self-hosting moves the control plane and the data onto your host; it does not move the compute. If you need an isolated topology, scope it with the Kortix team first.

## Update, back up, roll back

The updater runs daily, follows `stable` unless you say otherwise, migrates the database and rolls the services. It keeps the previous release, so a rollback is one command.

```bash
kortix self-host status
kortix self-host update --channel latest
kortix self-host update --tag <version>
kortix self-host rollback --release <version>
kortix self-host version
```

A pin holds through every daily run. To back up, stop the stack first so the database files are consistent. Copy the instance directory, then start again:

```bash
kortix self-host stop
sudo tar -czf kortix-backup.tgz -C ~/.config/kortix/self-host default
kortix self-host start
```

To restore, put the directory back in the same place and run `kortix self-host start`. `kortix self-host env ls --show` lists every setting, `kortix self-host logs kortix-api` follows one service's logs, and `kortix self-host ps` lists the services.

## Switch between your server and managed cloud

The CLI holds one host login at a time. Both hosts take the same commands, and `selfhost` appears as a host as soon as the stack starts.

```bash
kortix hosts ls
kortix hosts use selfhost
kortix hosts use cloud
kortix whoami
```

Managed cloud changes the host, not the repo. The manifest and the review path stay where they were. Run it on Kortix Cloud, in your VPC, or on your own on-prem network. Self-host is free.
