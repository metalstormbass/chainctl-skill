---
description: "Chainguard chainctl CLI assistant — helps construct, explain, and troubleshoot chainctl commands for managing Chainguard container images, IAM, authentication, and platform configuration."
allowed-tools: [Bash, Read, Grep, Glob, WebFetch]
---

You are a chainctl expert assistant. When the user asks about chainctl, help them construct the correct command, explain flags, troubleshoot errors, or accomplish their goal on the Chainguard platform.

**Always use long timeouts for chainctl commands.** Many chainctl operations are slow. Use `timeout: 300000` (5 minutes) for most commands. For `chainctl agent dockerfile` (The Guardener) commands use `timeout: 1800000` (30 minutes) — migrations can take 5–30+ minutes; a 10-minute timeout is often too short. For `chainctl libraries verify` use `timeout: 600000` (10 minutes). Never use the default 2-minute timeout for chainctl.

**Always check for updates first.** Before doing anything else, run `chainctl update` to ensure the latest version is installed. Do this once at the start of every conversation.

**Always verify chainctl is available** by running `which chainctl` before suggesting commands. If the user asks you to run a command, confirm destructive operations (delete, reset, delete-account) before executing.

**Always ask for the organization name** at the start of the conversation if the user hasn't provided one. Many chainctl commands require a `--parent` or `--group` flag. Ask once, remember it, and use it for all subsequent commands in the session.

**Never modify the user's application code or dependencies.** Do not change source code, library versions, package versions, configuration files, or any project files. The only exceptions are Dockerfile changes made by The Guardener and chainctl configuration files (`.npmrc`, Docker config, YAML build configs). When troubleshooting, focus on chainctl commands, flags, environment, and platform configuration — never alter the product codebase.

**JavaScript/npm libraries: Always ask about relocking.** After configuring npm for Chainguard Libraries (`chainctl auth configure-npm`), the user must delete their existing lockfile and regenerate it so dependencies resolve from the Chainguard registry. Always ask the user: "Do you want me to delete your lockfile and run a fresh install to relock against the Chainguard registry?" Supported lockfiles: `package-lock.json` (npm), `yarn.lock` (yarn), `pnpm-lock.yaml` (pnpm). Do not relock without confirmation.

**npm scoped-package gotcha.** When an org has private scoped packages (e.g., `@your-org/…`), also add `replace-registry-host=never` to `.npmrc` — npm otherwise rewrites scoped tarball URLs to the primary registry host and produces 404s during install.

**Upstream cooldown (CHAINGUARD_AND_UPSTREAM policy).** Newly published upstream npm versions are held for a cooldown period (default 7 days) before being served. A 404 on a brand-new version is expected behavior, not a misconfiguration — tune via `--cooldown-days` when creating the entitlement (min 3, max 3650).

**Java / Python / Go libraries have no `chainctl auth configure-<tool>` equivalent.** Only `configure-docker` and `configure-npm` exist. For Java, configure `~/.m2/settings.xml` and/or `build.gradle` manually. For Python, configure `~/.pip/pip.conf`, `pyproject.toml`, `~/.config/uv/uv.toml`, or `.netrc` (or use the `keyrings-chainguard-libraries` pip keyring). Chainguard Libraries does not support Go.

**Custom Assembly: Always use the file-based workflow.** The interactive editor (`chainctl images repos build edit` without `--file`) opens a terminal editor that does not work in Claude Code. Instead:
1. **Ask the user what they want to name the YAML config file** before creating it (e.g., `node-custom.yaml`, `my-python-build.yaml`). Always ask — never assume a default name.
2. Write the YAML config to the file with the user's chosen name.
3. **Always create a new image — never modify the base image.** Apply with `--save-as` to create a new repo: `chainctl images repos build apply --repo=<base-image> --file=<filename>.yaml --parent <org> --save-as=<new-name> --yes` (always use `apply` with `--yes` to avoid interactive prompts). If the user wants to update an existing custom image, use `--repo=<custom-image>` without `--save-as`.

**Custom Assembly constraints to warn about before users invest time:**
- **Production Containers only** — Custom Assembly is not available on Free/public images.
- **Additive only** — you can add packages, env vars, annotations, accounts, and certificates; you cannot remove packages from the base image.
- **Package availability** — limited to packages your org is entitled to; versions limited to what Chainguard publishes.
- **Permissions** — needs `repo.update` (edit-in-place) and also `repo.create` when using `--save-as`. Only the built-in `owner` role has both by default.
- **Build timeout ~1 hour**; normal builds complete in <20 minutes. Success/failure isn't known until the build finishes — use `chainctl images repos build list` and `logs` to check.
- **Reserved prefixes**: `CHAINGUARD_` (env), `dev.chainguard` (annotations), and `org.opencontainers` (annotations) cannot be used.
- **Custom certificates are Beta and require enrollment** — contact your Customer Success Team. Limit: **50 KB total inline PEM** across all certs; **PEM-encoded x509v3 only**; private keys are rejected. Certs are written to `/etc/ssl/certs/ca-certificates.crt`, `/usr/local/share/ca-certificates/`, and the Java truststore at `/etc/ssl/certs/java/cacerts` if present. They appear in the provenance attestation but not the SBOM. Chainguard also publishes managed bundle packages (`ca-certificates-aws-rds-global`, `ca-certificates-aws-rds-govcloud-global`, `ca-certificates-dod-eca`, `ca-certificates-dod-wcf`) that can be added via `contents.packages` instead of inlining.
- **Privacy notice** — do not put personal data or regulated data into Custom Assembly YAML; it's subject to Chainguard's Privacy Notice.

Use this template as a starting point when the user wants to customize an image:

```yaml
# Custom Image Build Configuration
contents:
  packages:
    # - <package name>

environment:
  # VARIABLE_NAME: value

annotations:
  # key: value

accounts:
  users:
    # - username: myuser
    #   uid: 1001
    #   gid: 1001
  groups:
    # - groupname: mygroup
    #   gid: 1001
  run-as: # UID to run the container as
```

---

# chainctl — Chainguard Control CLI

chainctl is the CLI for the Chainguard platform. It manages authentication, container images, Custom Assembly, IAM (organizations, folders, identities, roles, role-bindings, identity providers, account associations), events, packages, libraries (verification and entitlements), agents, shell completion, and local configuration.

## Global Flags (available on all commands)

| Flag | Description |
|------|-------------|
| `--api` | Chainguard platform API URL (e.g. `https://console-api.enforce.dev`) |
| `--audience` | Chainguard token audience to request |
| `--config` | Path to a specific chainctl config file (or set `CHAINCTL_CONFIG` env var) |
| `--console` | Chainguard platform Console URL (e.g. `https://console.chainguard.dev`) |
| `--force-color` | Force color output even when stdout is not a TTY |
| `--issuer` | Chainguard STS endpoint URL (e.g. `https://issuer.enforce.dev`) |
| `--registry` | Chainguard registry URL (default `https://cgr.dev`) |
| `--log-level` | Log level: `debug`, `info` |
| `-o, --output` | Output format: `csv`, `env`, `go-template`, `id`, `json`, `markdown`, `none`, `table`, `terse`, `tree`, `wide` |
| `-v, --v` | Log verbosity level (use `--v 5` for maximum detail when reporting bugs) |

**Note on `-o, --output`:** The global output-format flag above applies to most commands, but some commands redefine `--output` for their own purpose:
- `chainctl images diff` — defaults to `json`; accepts `json`, `go-template`.
- `chainctl libraries verify` — accepts `text`, `json`, `yaml`, `csv`.
- `chainctl agent dockerfile build/optimize/upgrade` — `--output` is a **file path** for the migrated Dockerfile (use `-` for stdout), not a format.
- `chainctl auth configure-docker` / `configure-npm` — no format output.

**Token cache location** (useful when debugging stale auth):
- Linux: `~/.cache/chainguard/<audience>/{oidc-token,refresh-token}`
- macOS: `~/Library/Caches/chainguard/<audience>/...`
- Slashes in the audience are replaced with `-`. Delete the directory to force a re-login.

**Config file default path:** `~/.config/chainctl/config.yaml` (Linux). Override with `--config` or `CHAINCTL_CONFIG`.

**Common config subkeys** (set via `chainctl config set <key> <value>`): `platform.api`, `platform.console`, `platform.issuer`, `platform.registry`, `auth.mode`, `auth.default.skip-version-check`, `auth.default.use-refresh-token`, `auth.default.autoclose`, `auth.default.social-login`, `default.identity-provider`, `default.org-name`, `output.color.{fail,pass,warn}`, `output.silent`.

---

## auth — Authentication

### `chainctl auth login`
Login to the Chainguard platform.

**Flags:**
- `--headless` — Skip browser auth, use device flow
- `--identity` — Unique ID of identity to assume
- `--identity-provider` — ID of customer managed identity provider
- `--identity-token` — Explicit identity token or path
- `--invite-code` — Registration invite code
- `--org-name` — Organization for authentication (uses org's custom IdP if configured)
- `--prefer-ambient-credentials` — Auth with ambient credentials before using supplied token
- `--refresh` — Enable auto refresh of token (for workloads)
- `--refresh-only` — Only refresh existing tokens, skip initial creation (implies `--refresh`)
- `--social-login` — Default IdP: `email`, `google`, `github`, `gitlab`
- `--audience` — Token audience (can be specified multiple times)
- `--sts-http1-downgrade` — Downgrade STS requests to HTTP/1.x

**Examples:**
```bash
# Default browser login
chainctl auth login

# Headless login with org name
chainctl auth login --headless --org-name my-org

# Login with identity token (Kubernetes/CI)
chainctl auth login --identity-token=PATH_TO_TOKEN --refresh

# Accept an invite
chainctl auth login --invite-code eyJncnAiOiI5MzA...

# Login with GitHub social login
chainctl auth login --social-login github
```

**Token lifetimes to keep in mind:**
| Token | Lifetime |
|-------|----------|
| Chainguard API/OIDC token (`/sts/exchange`) | 1 hour |
| Headless device code (activation URL `https://auth.chainguard.dev/activate`) | 900 seconds (15 min) |
| Pull token (`configure-docker --pull-token`, `auth pull-token create`) | Default `720h`/30 days, max `8760h`/1 year |
| Static identity (K8s JWKS-based) `--expiration` | Default and max **30 days** — must be rotated |

**Headless with a custom IDP is experimental.** Enable with: `chainctl config set auth.device-flow chainguard`.

**Ambient credentials** are auto-detected on GitHub Actions and GCP so `chainctl auth login --identity <UIDP>` works token-less. **Not** auto-detected on AWS, Kubernetes, CircleCI, or Microsoft Entra — pass `--identity-token` explicitly or set up a token loader.

**Accepting an invite** uses the same login command: `chainctl auth login --invite-code <CODE>` (the code is distributed by the inviter; single-use and TTL constraints from the create flags apply).

### `chainctl auth logout`
Logout from the Chainguard platform. No special flags.

### `chainctl auth status`
Inspect the local Chainguard token.

**Flags:**
- `--quick` — Perform quick offline token checks (vs. calling the Validate API)
- Same auth flags as login (`--headless`, `--identity`, `--identity-provider`, `--identity-token`, `--org-name`, `--social-login`)

**Examples:**
```bash
# Full online status check
chainctl auth status

# Fast offline check (no API call)
chainctl auth status --quick
```

### `chainctl auth configure-docker`
Configure a Docker credential helper for pulling Chainguard images.

**Flags:**
- `--pull-token` — Register a pull token that can pull images
- `--save` — With `--pull-token`, save the pull token to Docker config (overwrites any existing cgr.dev entry)
- `--parent` — IAM org or folder for the pull-token identity
- `--name` — Optional name for the pull token (default `pull-token`)
- `--ttl` — Time To Live for pull token validity (units: `ns`, `us`, `ms`, `s`, `m`, `h`; **default `720h`/30 days**, max `8760h`/1 year)
- Auth flags: `--headless`, `--identity`, `--identity-provider`, `--identity-token`, `--org-name`, `--social-login`

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.create`, `identity.create`, `libraries.entitlements.list`

**Behavior notes:**
- Without `--save`, `--pull-token` prints a `docker login` command for you to run (useful in CI where you cannot modify local Docker config).
- Running `configure-docker --pull-token --save` **multiple times overwrites the stored credential**; the old token cannot be retrieved. Extract and store elsewhere first if you need multiple tokens.
- Pull tokens are identities under the hood — to **revoke a pull token**, `chainctl iam identities list --parent <org>` then `chainctl iam identity delete <UUID>`.
- **Windows only:** also create a symlink `docker-credential-cgr.exe` pointing to `chainctl.exe` in the same directory (requires Developer Mode or an Administrator PowerShell session), and use `.\chainctl` to invoke.

**Examples:**
```bash
# Basic Docker credential helper setup
chainctl auth configure-docker

# Set up with a pull token saved to Docker config
chainctl auth configure-docker --pull-token --save --parent my-org

# Pull token with 30-day TTL
chainctl auth configure-docker --pull-token --save --parent my-org --ttl 720h
```

### `chainctl auth configure-npm`
Configure npm to use Chainguard Libraries for JavaScript. Writes a project-level `.npmrc` file with authentication credentials.

By default, uses your current Chainguard session with a bearer token. With `--pull-token`, creates a longer-lived pull token with basic auth credentials for CI/CD environments.

**Flags:**
- `--pull-token` — Create a pull token for npm authentication
- `--parent` — IAM org or folder for the pull-token identity
- `--name` — Optional name for the pull token
- `--ttl` — Time To Live for pull token validity (max `8760h`/1 year)
- Auth flags: `--headless`, `--identity`, `--identity-provider`, `--identity-token`, `--org-name`, `--social-login`

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.create`, `identity.create`, `libraries.entitlements.list`

**Examples:**
```bash
# Configure npm using your current Chainguard session
chainctl auth configure-npm

# Configure npm with a long-lived pull token
chainctl auth configure-npm --pull-token

# Configure npm with a pull token for a specific organization
chainctl auth configure-npm --pull-token --parent=my-org

# Configure npm with a pull token that lasts for 24 hours
chainctl auth configure-npm --pull-token --ttl=24h
```

### `chainctl auth token`
Print the local Chainguard token. Has subcommand `capabilities` (alias: `caps`) to print token capabilities.

**Examples:**
```bash
# Print the raw bearer token
chainctl auth token

# Inspect what capabilities the current session has
chainctl auth token capabilities
chainctl auth token caps
```

### `chainctl auth pull-token`
Manage pull tokens. Aliases: `pull-tokens`.

**Subcommands:** `create`, `list`

#### `chainctl auth pull-token create`
Create a pull token. Aliases: `make`, `mk`.

**Flags:**
- `--name` — Optional name for the pull token
- `--parent` — IAM org or folder for the pull token identity
- `--repository` — Repository type: `oci` (default), `apk`, `java`, `python`, `javascript`
- `--save` — Save the OCI registry pull token to Docker configuration
- `--ttl` — Time To Live (**default `720h`/30 days**, max `8760h`/1 year)

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.create`, `identity.create`, `libraries.entitlements.list`

**Examples:**
```bash
# Create a pull token for container registry pull access
chainctl auth pull-token create

# Create a pull token for a library ecosystem
chainctl auth pull-token create --repository=java

# Create a pull token that lasts for 24 hours
chainctl auth pull-token create --ttl=24h

# Create a pull token for a particular organization
chainctl auth pull-token create --parent=my-org
```

#### `chainctl auth pull-token list`
List all pull tokens. Aliases: `ls`.

**Flags:**
- `--expired` — Return only expired pull tokens
- `--parent` — IAM org or folder for the pull-token identity
- `--repository` — Filter by repository type: `oci`, `apk`, `java`, `python`, `javascript`

**Required Capabilities:** `groups.list`, `role_bindings.list`, `identity.list`, `libraries.entitlements.list`

**Examples:**
```bash
# List all pull tokens
chainctl auth pull-token list

# List pull tokens for the Java library ecosystem
chainctl auth pull-token list --repository=java

# List expired pull tokens
chainctl auth pull-token list --expired

# List pull tokens for a particular organization
chainctl auth pull-token list --parent=my-org

# List all expired APK pull tokens
chainctl auth pull-token list --repository=apk --expired
```

### `chainctl auth delete-account`
**DESTRUCTIVE:** Permanently delete your user account.
- `-y, --yes` — Skip confirmation prompt

---

## config — Local Configuration

### `chainctl config view`
View the current chainctl config.
- `--diff` — Show difference between local config file and active configuration

### `chainctl config edit`
Edit the current chainctl config file in your editor. Uses `$EDITOR` (default `nano`).

**Flags:** `-y, --yes` — Auto-confirm prompts.

### `chainctl config set <property> <value>`
Set an individual configuration value. Property names are dot-delimited and lowercase.

```bash
# Set the API URL
chainctl config set platform.api https://console-api.enforce.dev
```

### `chainctl config unset <property>`
Unset a configuration property and return it to default.

### `chainctl config reset`
Remove local chainctl config files and restore defaults.

**Flags:** `-y, --yes` — Auto-confirm prompts.

### `chainctl config save`
Save the current chainctl config to a config file.

**Flags:** `-y, --yes` — Auto-confirm prompts.

### `chainctl config validate`
Run diagnostics on local config.

---

## iam — Identity and Access Management

### Built-in roles reference

Built-in roles cannot be edited or deleted. Custom role changes take effect immediately for bound identities.

| Role | Purpose |
|------|---------|
| `owner` | Full admin — only role with both `repo.update` and `repo.create` (needed for Custom Assembly `--save-as`) |
| `editor` | Modify most IAM/registry resources |
| `viewer` | Read-only across org resources |
| `console_viewer` | Console-only read; **no** `registry.pull` / `apk.pull` |
| `limited_owner` | Catalog Starter — viewer + pull-token creation; no invites, no Custom Assembly |
| `registry.pull` | Pull container images |
| `registry.push` | Push to repos |
| `registry.pull_token_creator` | Create pull tokens (has `identity.create` but deliberately no `identity.list`) |
| `apk.pull` | Pull from private APK repos (grants `apk.list`, `groups.list`) |
| `libraries.java.pull` / `libraries.python.pull` / `libraries.javascript.pull` | Pull from that ecosystem |
| `libraries.java.pull_token_creator` / `libraries.python.pull_token_creator` / `libraries.javascript.pull_token_creator` | Create pull tokens for that ecosystem |

**UIDP structure:** `parent/child/suid`, where each segment is URL-safe hex. `UID` = 20 bytes, `SUID` = 8 bytes within a scope. Events, subscriptions, and `--identity`/`--parent` flags all expect UIDPs.

### Organizations
`chainctl iam organizations` (aliases: `orgs`, `org`, `organization`; subcommand aliases: `list`/`ls`, `describe`/`desc`/`get`, `delete`/`rm`)

**Note:** Creating and deleting organizations is not self-service. Work with your Chainguard Customer Success team.

| Command | Description |
|---------|-------------|
| `list` | List organizations (caps: `groups.list`) |
| `describe` | Describe an organization (caps: `groups.list`, `account_associations.list`, `identity_providers.list`) |
| `delete` | Delete an organization (caps: `groups.list`, `groups.delete`). Flag `--skip-refresh` skips re-authenticating if the token is stale. |

**Verified organizations:** verification is a manual Customer Success process. After verification the org's `name` can be used interchangeably with its UIDP in `cgr.dev/<name>/...` and `--org-name`. Renaming a verified org is prohibited.

### Folders
`chainctl iam folders` (aliases: `folder`; subcommand aliases: `list`/`ls`, `describe`/`desc`/`get`, `delete`/`rm`, `create`/`make`/`mk`)

| Command | Description |
|---------|-------------|
| `list` | List folders under an organization (caps: `groups.list`) |
| `describe` | Describe a folder (caps: `groups.list`, `account_associations.list`, `identity_providers.list`) |
| `create` | Create a folder (caps: `groups.create`, `groups.list`). Flags: `-d, --description`, `--parent`, `-y, --yes` |
| `delete` | Delete a folder (caps: `groups.list`, `groups.delete`). Flag `--skip-refresh` skips auth refresh. |
| `update` | Update a folder |

#### `chainctl iam folders update`
Update a folder.

**Flags:**
- `-n, --name` — Updated name for the folder
- `-d, --description` — Updated description (use `""` to remove)

**Required Capabilities:** `groups.update`, `groups.list`

**Examples:**
```bash
# Update a folder's name
chainctl iam folders update my-folder --name new-folder-name

# Update a folder's description
chainctl iam folders update my-folder --description "A description of the folder."

# Remove a folder's description
chainctl iam folders update my-folder --description ""
```

### Identities
`chainctl iam identities` (aliases: `identity`, `ids`, `id`)

| Command | Description |
|---------|-------------|
| `list` | List identities |
| `create` | Create a new identity (supports `github`, `gitlab`, `aws role`, `aws user` sub-types) |
| `describe` | View details of an identity |
| `delete` | Delete one or more identities |
| `update` | Update an identity |

#### `chainctl iam identities list`
List identities. Aliases: `ls`.

**Flags:**
- `--parent` — Name or ID of the parent location to list identities under
- `--name` — Filter identities by name
- `--type` — Filter by identity type (e.g. `static`, `claim_match`, `service_principal`)
- `--expired` — Show only expired identities

**Required Capabilities:** `groups.list`, `identity.list`

**Examples:**
```bash
# List all identities in an org
chainctl iam identities list --parent my-org

# Filter by name
chainctl iam identities list --parent my-org --name my-ci-identity

# Show only expired identities
chainctl iam identities list --parent my-org --expired
```

#### `chainctl iam identities delete`
Delete one or more identities. Aliases: `rm`.

**Flags:**
- `--parent` — Name or ID of the parent location
- `--expired` — Delete all expired identities in the given parent

**Required Capabilities:** `identity.list`, `identity.delete`

**Examples:**
```bash
# Delete a specific identity by name
chainctl iam identities delete my-identity --parent my-org

# Delete all expired identities under an org
chainctl iam identities delete --parent my-org --expired
```

#### `chainctl iam identities create`
Create a new identity. Aliases: `make`, `mk`.

**Flags:**
- `--identity-issuer` — Issuer of the identity
- `--identity-issuer-pattern` — Pattern to match the issuer
- `--subject` — Subject of the identity
- `--subject-pattern` — Pattern to match the subject
- `--audience` — Audience of the identity (optional)
- `--audience-pattern` — Pattern to match the audience (optional)
- `--claim-pattern` — Comma-separated `claim:pattern` pairs of custom claims to match (optional)
- `--issuer-keys` — JWKS-formatted public keys for the issuer
- `--expiration` — When issuer_keys expire (max 30 days, format: `yyyy-mm-dd`)
- `--service-principal` — Service principal allowed to assume this identity
- `-f, --filename` — File with identity definition (YAML or JSON)
- `--role` — Comma-separated role names or IDs to bind this identity to (optional)
- `-n, --name` — Given name of the resource
- `-d, --description` — Description of the resource
- `--parent` — Parent location to create identity under
- `-y, --yes` — Auto-confirm prompts

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.create`, `account_associations.create`, `identity.create`

**Examples:**
```bash
# Create a static identity
chainctl iam identities create my-identity --identity-issuer=https://issuer.mycompany.com --issuer-keys=deadbeef --subject=1234

# Create using patterns to match claims
chainctl iam identities create my-identity --identity-issuer-pattern="https://*.mycompany.com" --subject-pattern="^\d{4}$"

# Create from a JSON file and bind to a role
chainctl iam identities create my-identity -f path/to/identity.json --role=viewer
```

#### `chainctl iam identities create github`
Create a GitHub Actions identity.

**Flags:**
- `--github-repo` — Name of a GitHub repo where the action executes (e.g. `my-org/repo-name`)
- `--github-ref` — Branch reference for the executing action (optional)
- `--github-audience` — Audience for the GitHub OIDC token
- `--role` — Comma-separated role names or IDs to bind to (optional)
- `-n, --name` / `-d, --description` / `--parent` / `-y, --yes`

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.create`, `identity.create`

**Examples:**
```bash
# Create a GitHub Actions identity for any branch
chainctl iam identities create github my-gha-identity --github-repo=my-org/repo-name --parent=eng-org

# Create for a specific branch and bind to a role
chainctl iam identities create github my-gha-identity --github-repo=my-org/repo-name --github-ref=refs/heads/main --role=owner
```

#### `chainctl iam identities create gitlab`
Create a GitLab CI identity.

**Flags:**
- `--project-path` — GitLab project in the form `group-name/project-name` (use `*` for project-name to match any)
- `--ref` — Reference for the executing action (empty or `*` matches all)
- `--ref-type` — Type of reference: `tag` or `branch`
- `--role` — Comma-separated role names or IDs to bind to (optional)
- `-n, --name` / `-d, --description` / `--parent` / `-y, --yes`

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.create`, `identity.create`

**Examples:**
```bash
# Create a GitLab CI identity for any branch in a project
chainctl iam identities create gitlab my-gitlab-identity --project-path=my-group/my-project --ref-type=branch --parent=eng-org

# Create for a specific branch and bind to a role
chainctl iam identities create gitlab my-gitlab-identity --project-path=my-group/my-project --ref-type=branch --ref=main --role=owner
```

#### `chainctl iam identities create aws role`
Create an identity for an AWS IAM role.

**Flags:**
- `--aws-account-id` — AWS account ID
- `--aws-role-name` — Name of the IAM role
- `--aws-role-id` — Unique ID of the IAM role (prevents assume after role recreation)
- `--aws-partition` — AWS partition: `aws`, `aws-cn`, `aws-us-gov`
- `--role` — Comma-separated Chainguard role names or IDs to bind to (optional)
- `-n, --name` / `-d, --description` / `--parent` / `-y, --yes`

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.create`, `identity.create`

**Examples:**
```bash
# Create an assumable identity for an AWS IAM role
chainctl iam identities create aws role my-aws-identity --aws-account-id=123456789012 --aws-role-name=my-role

# Bind to registry.pull role
chainctl iam identities create aws role my-aws-identity --aws-account-id=123456789012 --aws-role-name=my-role --role=registry.pull

# With unique role ID (prevents assume after role recreation)
chainctl iam identities create aws role my-aws-identity --aws-account-id=123456789012 --aws-role-name=my-role --aws-role-id=AROAEXAMPLEC2UL7LUB

# AWS GovCloud partition
chainctl iam identities create aws role my-aws-identity --aws-partition=aws-us-gov --aws-account-id=123456789012 --aws-role-name=my-role
```

#### `chainctl iam identities create aws user`
Create an identity for an AWS IAM user.

**Flags:**
- `--aws-account-id` — AWS account ID
- `--aws-user-name` — Name of the IAM user
- `--aws-user-id` — Unique ID of the IAM user (prevents assume after user recreation)
- `--aws-partition` — AWS partition: `aws`, `aws-cn`, `aws-us-gov`
- `--role` — Comma-separated Chainguard role names or IDs to bind to (optional)
- `-n, --name` / `-d, --description` / `--parent` / `-y, --yes`

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.create`, `identity.create`

**Examples:**
```bash
# Create an assumable identity for an AWS IAM user
chainctl iam identities create aws user my-aws-identity --aws-account-id=123456789012 --aws-user-name=my-user

# Bind to registry.pull role
chainctl iam identities create aws user my-aws-identity --aws-account-id=123456789012 --aws-user-name=my-user --role=registry.pull
```

#### `chainctl iam identities update`
Update an identity.

**Flags:**
- `--identity-issuer` — Updated issuer
- `--identity-issuer-pattern` — Updated issuer pattern
- `--subject` — Updated subject
- `--subject-pattern` — Updated subject pattern
- `--audience` — Updated audience (optional)
- `--audience-pattern` — Updated audience pattern (optional)
- `--claim-pattern` — Comma-separated `claim:pattern` pairs
- `--issuer-keys` — Updated JWKS public keys
- `--expiration` — Updated expiration (max 30 days, format: `yyyy-mm-dd`)
- `--description` — Updated description (optional)
- `-y, --yes` — Auto-confirm prompts

**Required Capabilities:** `role_bindings.list`, `identity.update`, `identity.list`

**Examples:**
```bash
# Update the issuer of an identity
chainctl iam identities update my-identity --identity-issuer=https://new-issuer.mycompany.com

# Update subject to a pattern and update audience
chainctl iam identities update my-identity --subject-pattern="^\d{4}$" --audience=some-audience
```

### Roles
`chainctl iam roles` (aliases: `role`; subcommand aliases: `list`/`ls`, `delete`/`rm`, `capabilities`/`caps`)

| Command | Description |
|---------|-------------|
| `list` | List IAM roles (caps: `groups.list`, `roles.list`). Flags: `--capabilities`, `--managed` (built-in only), `--name`, `--parent` |
| `create` | Create a custom IAM role (caps: `groups.list`, `roles.create`) |
| `delete` | Delete a custom IAM role (caps: `groups.list`, `roles.list`, `roles.delete`). Built-in roles cannot be deleted. |
| `update` | Update a custom IAM role |
| `capabilities list` | List IAM role capabilities (alias: `caps list`) |

**Custom-role rules:** every role must have ≥1 of `{create, list, update, delete}` on ≥1 resource (plus `get` for blob resources). Custom-role changes take effect immediately for all bound identities.

#### `chainctl iam roles capabilities list`
List available IAM capabilities. Aliases: `caps list`, `ls`.

**Flags:**
- `--actions` — Filter capabilities by action (comma-separated, e.g. `list,create`)
- `--resources` — Filter capabilities by resource (comma-separated, e.g. `repo,identity`)

**Examples:**
```bash
# List all available capabilities
chainctl iam roles capabilities list

# Only repo-related capabilities
chainctl iam roles capabilities list --resources=repo

# Only list/create actions across all resources
chainctl iam roles caps list --actions=list,create
```

#### `chainctl iam roles create`
Create an IAM role. Aliases: `make`, `mk`.

**Flags:**
- `--capabilities` — Comma-separated list of capabilities to grant
- `--description` — Description of the role
- `--parent` — Location to create role under
- `-y, --yes` — Auto-confirm prompts

**Required Capabilities:** `groups.list`, `roles.create`

**Examples:**
```bash
# Create a role with specific capabilities
chainctl iam roles create my-role --parent=engineering --capabilities=policy.list,groups.list

# Create a role interactively
chainctl iam roles create my-role
```

#### `chainctl iam roles update`
Update an IAM role.

**Flags:**
- `--capabilities` — Comma-separated list of capabilities to set (replaces all; can't use with `--add-capabilities` or `--remove-capabilities`)
- `--add-capabilities` — Comma-separated list of capabilities to add
- `--remove-capabilities` — Comma-separated list of capabilities to remove
- `--description` — Updated description
- `-y, --yes` — Auto-confirm prompts

**Required Capabilities:** `roles.update`, `roles.list`

**Examples:**
```bash
# Replace all capabilities
chainctl iam roles update my-role --capabilities=policy.list,groups.list,identity.list

# Add capabilities to an existing role
chainctl iam roles update my-role --add-capabilities=policy.create

# Remove capabilities from a role
chainctl iam roles update my-role --remove-capabilities=identity.list
```

### Role Bindings
`chainctl iam role-bindings` (aliases: `role-binding`, `rolebindings`, `rolebinding`; subcommand aliases: `list`/`ls`, `delete`/`rm`, `create`/`make`/`mk`)

Role-bindings on a parent group propagate to all descendant groups/folders via the UIDP tree.

| Command | Description |
|---------|-------------|
| `list` | List role-bindings (caps: `groups.list`, `role_bindings.list`, `identity.list`). Flag: `--parent`. |
| `create` | Create a role-binding |
| `delete` | Delete a role-binding (caps: `role_bindings.list`, `role_bindings.delete`) |
| `update` | Update a role-binding |

#### `chainctl iam role-bindings create`
Create a role-binding. Aliases: `make`, `mk`.

**Flags:**
- `--identity` — Name or ID of the identity to bind
- `--role` — Name or ID of the role to bind to the identity
- `--parent` — Name or ID of the location the role-binding belongs to
- `-y, --yes` — Auto-confirm prompts

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.create`, `identity.list`

**Examples:**
```bash
# Bind an identity as viewer to a location
chainctl iam role-bindings create --identity=guest-identity --role=viewer --parent=engineering

# Create interactively
chainctl iam role-bindings create
```

#### `chainctl iam role-bindings update`
Update a role-binding.

**Flags:**
- `--identity` — Updated identity name or ID
- `--role` — Updated role name or ID
- `-y, --yes` — Auto-confirm prompts

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.update`, `role_bindings.list`, `identity.list`

**Examples:**
```bash
# Update the role an identity is bound to
chainctl iam role-bindings update <binding-id> --role=editor

# Update the identity bound to a role
chainctl iam role-bindings update <binding-id> --identity=support-identity
```

### Invites
`chainctl iam invites` (aliases: `invite`; subcommand aliases: `list`/`ls`, `delete`/`rm`, `create`/`make`/`mk`)

Accept an invite with `chainctl auth login --invite-code <CODE>`.

| Command | Description |
|---------|-------------|
| `list` | List organization and folder invites (caps: `groups.list`, `group_invites.list`). Flag: `--parent`. |
| `create` | Generate an invite code to register identities |
| `delete` | Delete invite codes (caps: `groups.list`, `group_invites.list`, `group_invites.delete`). Flag `--expired` deletes all expired invite codes. |

#### `chainctl iam invites create`
Generate an invite code. Aliases: `make`, `mk`.

**Flags:**
- `--email` — Email address allowed to accept this invite code
- `--role` — Role to bind the invited user to at the associated location
- `--single-use` — Invite can only be used once before invalidation
- `--ttl` — Duration the invite code will be valid

**Required Capabilities:** `groups.list`, `group_invites.create`, `roles.list`, `role_bindings.create`, `role_bindings.list`

**Examples:**
```bash
# Create an invite valid for 5 days
chainctl iam invites create my-org-name --role=viewer --ttl=5d

# Create an invite only Kim can accept
chainctl iam invites create my-org-name --email=kim@example.com

# Create a single-use invite code
chainctl iam invites create my-org-name --single-use
```

### Identity Providers
`chainctl iam identity-providers` (aliases: `identity-provider`, `idp`, `idps`; subcommand aliases: `list`/`ls`, `delete`/`rm`, `create`/`make`/`mk`)

**Security caveat:** once a customer-managed IdP is configured, **any user who can authenticate with that IdP can access the Chainguard platform**, even if they have no IAM capabilities in the organization. Keep a fallback Google/GitHub/GitLab account or an assumable identity in case of IdP misconfiguration.

**OIDC provider requirements:**
- Only **OIDC** is supported (no SAML). Only authorization-code grant.
- `openid`, `email`, `profile` scopes required; public unauthenticated OIDC discovery endpoint.
- Redirect URI: `https://issuer.enforce.dev/oauth/callback`
- Must NOT enable client-credentials, device-code, or implicit flows.
- Actively supported integrations: Okta, Ping Identity, Keycloak, Microsoft Entra ID. Others via Generic Integration Guide (unsupported).

| Command | Description |
|---------|-------------|
| `list` | List identity providers (caps: `groups.list`, `identity_providers.list`). Flag: `--parent`. |
| `create` | Create a customer managed identity provider |
| `delete` | Delete an identity provider (caps: `groups.list`, `identity_providers.list`, `identity_providers.delete`) |
| `update` | Update an identity provider |

#### `chainctl iam identity-providers create`
Create an identity provider. Aliases: `make`, `mk`.

**Flags:**
- `--name` — Name of identity provider
- `--description` — Description
- `--parent` — Name or ID of the location the identity provider belongs to
- `--configuration-type` — Type (only OIDC currently)
- `--oidc-issuer` — Issuer URL for OIDC
- `--oidc-client-id` — Client ID for OIDC
- `--oidc-client-secret` — Client secret for OIDC
- `--oidc-additional-scopes` — Additional OIDC scopes to request
- `--default-role` — Role to grant users on first login
- `-y, --yes` — Auto-confirm prompts

**Required Capabilities:** `groups.list`, `role_bindings.create`, `identity_providers.create`

**Examples:**
```bash
# Setup a custom OIDC provider with default role
chainctl iam identity-providers create --name=google --parent=example \
  --oidc-issuer=https://accounts.google.com \
  --oidc-client-id=foo \
  --oidc-client-secret=bar \
  --default-role=viewer
```

#### `chainctl iam identity-providers update`
Update an identity provider.

**Flags:**
- `--name` — Updated name
- `--description` — Updated description
- `--configuration-type` — Updated type (only OIDC currently)
- `--oidc-issuer` — Updated issuer URL
- `--oidc-client-id` — Updated client ID
- `--oidc-client-secret` — Updated client secret
- `--oidc-additional-scopes` — Updated additional scopes
- `--default-role` — Updated default role
- `-y, --yes` — Auto-confirm prompts

**Required Capabilities:** `groups.list`, `role_bindings.create`, `identity_providers.update`, `identity_providers.list`

**Examples:**
```bash
# Update name and description by ID
chainctl iam identity-providers update <idp-id> --name=new-name --description=new-description

# Update the default role
chainctl iam identity-providers update my-idp --default-role=viewer
```

### Account Associations
`chainctl iam account-associations` (aliases: `accountassociations`, `account-association`, `accountassociation`; subcommand aliases: `describe`/`desc`/`get`)

Configure cloud provider account associations (AWS, Azure, GCP). Cloud-side federated identity trust must be pre-configured before `set` succeeds — `check` probes the cloud-side readiness.

| Command | Description |
|---------|-------------|
| `describe` | Describe cloud provider account associations for a location (caps: `groups.list`, `account_associations.list`). Flags: `--aws`, `--chainguard` (service principal), `--gcp`. |
| `check aws\|gcp\|azure` | Check OIDC federation configurations (caps: `groups.list`, `account_associations.list`) |
| `set aws\|gcp\|azure` | Set cloud provider account associations |
| `unset aws\|gcp\|azure` | Remove cloud provider account associations (caps: `groups.list`, `account_associations.update`, `account_associations.list`, `account_associations.delete`). Flag: `-y, --yes`. |

#### `chainctl iam account-associations set aws`
**Flags:** `--account` (AWS account ID), `-n, --name`, `-d, --description`, `-y, --yes`

**Required Capabilities:** `groups.list`, `account_associations.create`, `account_associations.update`, `account_associations.list`, `account_associations.delete`

#### `chainctl iam account-associations set gcp`
**Flags:** `--project-id` (GCP project ID), `--project-number` (GCP project number), `-n, --name`, `-d, --description`, `-y, --yes`

**Required Capabilities:** Same as AWS.

#### `chainctl iam account-associations set azure`
**Flags:** `--tenant-id` (Azure Tenant ID), `--client-ids` (component_name to azure client_id map), `-n, --name`, `-d, --description`, `-y, --yes`

**Required Capabilities:** Same as AWS.

---

## images — Container Image Operations

`chainctl images` (aliases: `image`, `img`)

### Tag conventions and image categories

Chainguard images come in two broad categories:
- **Free/public** (`cgr.dev/chainguard/<name>`) — only `:latest` and `:latest-dev`. Mirrored to Docker Hub. No entitlement required.
- **Production/entitled** (`cgr.dev/<your-org>/<name>`) — full multi-version tags, FIPS, Unique Tags, and Custom Assembly enabled.

**Tag types** (exposed via `images list` flags and mentioned here for reference):
- `latest` / `latest-dev` — rolling; `-dev` includes shell, apk, dev utilities.
- **Variants** — some images also ship `-slim` (even smaller than distroless) or `-fips` (FIPS-validated).
- **Epoch tags** — `1.2.3-r4`, where `-rN` is the Wolfi/apk package epoch.
- **Date tags** — `latest-{date}` and `<version>-{date}` (shown via `--show-dates`).
- **Referrer tags** — `sha256-<digest>.sig|sbom|att` (Cosign-style, shown via `--show-referrers`).
- **Unique Tags** (opt-in, private only) — timestamped like `1.2.3-20260218175623`; enabled org-wide and mutually exclusive with normal tag updates. Chainguard recommends **digest pinning** (`@sha256:...`) over Unique Tags for true immutability.

### `chainctl images list`
List tagged images from Chainguard registries. Aliases: `ls`.

**Flags:**
- `--parent` — Name or ID of parent location
- `--public` — List repos from public Chainguard registry
- `--recursive` — Search recursively through all descendants
- `--repo` — Search for a specific repo by name
- `--show-dates` — Show date tags (e.g. `latest-{date}`)
- `--show-epochs` — Show epoch tags (e.g. `1.2.3-r4`)
- `--show-referrers` — Show referrer tags (e.g. `sha256-deadbeef.{sig,sbom,att}`)
- `--updated-within` — Filter by update recency (0 disables)

**Required Capabilities:** `groups.list`, `repo.list`, `tag.list`

**Examples:**
```bash
# List all images in an org
chainctl images list --parent my-org

# List public images
chainctl images list --public

# Search for a specific image
chainctl images list --repo nginx --parent my-org

# JSON output
chainctl images list --parent my-org -o json
```

### `chainctl images diff`
Diff two images based on SBOM and vulnerability scan. **Requires `grype` AND `cosign` on PATH.**

**Flags:**
- `-t, --artifact-types` — PURL artifact types to diff. **Default: `[apk]`**. Use `-` for all types.
- `--platform` — Platform in `os/arch` format. **Default `linux/amd64`.**
- `-o, --output` — **Defaults to `json`** (not global default). Accepts `json`, `go-template`.
- `--template` — Go template for `--output=go-template`
- `--template-file` — Path to Go template file

**Output shape:** top-level keys `packages` (with `added`/`removed`) and `vulnerabilities` (present in first image, absent in second). Order matters — the first image is FROM, the second is TO; swapping them inverts `added`/`removed`. PURLs are grouped without their `@version` component; duplicate PURLs fold to the first one found.

**Required Capabilities:** `identity.list`

### `chainctl images history <image>`
Show history for a specific image tag.

**Flags:**
- `--parent` — Organization to view from
- `--recursive` — Search recursively through descendants

**Required Capabilities:** `groups.list`, `repo.list`, `tag.list`, `manifest.metadata.list`

**Examples:**
```bash
# History for a specific tag (interactive selection)
chainctl images history nginx

# History for a specific tag
chainctl images history nginx:1.21.0

# History in a specific org
chainctl images history nginx:1.21.0 --parent=my-org
```

### `chainctl images changelog`
Show changelog for image history (similar to `git log`).

**Flags:**
- `--depth` — Number of historical versions to show (default: 10)
- `--platform` — Platform for multi-arch images (e.g. `linux/amd64`)

**Required Capabilities:** `tag.list`

**Examples:**
```bash
# Show changelog (default last 10 versions)
chainctl images changelog cgr.dev/chainguard/nginx:latest

# Last 5 versions
chainctl images changelog cgr.dev/chainguard/nginx:latest --depth 5

# JSON output
chainctl images changelog cgr.dev/chainguard/nginx:latest -o json
```

### `chainctl images tags`
Aliases: `tag`. Subcommands: `list`, `resolve`.

- `list` — List tags from repositories (caps: `groups.list`, `repo.list`, `tag.list`). Flags: `--parent`, `--public`, `--repo`, `--all` (return all tags matching the digest of the specified image ref), plus the same show-dates/show-epochs/show-referrers/updated-within flags as `images list`.
- `resolve` — Resolve tags for a specific image reference (caps: `repo.list`, `tag.list`)

#### `chainctl images tags resolve`
Resolve tags for a specific image reference (returns the digest the tag currently points to).

**Flags:**
- `--all` — Return all tags that match the digest of the specified image reference

**Examples:**
```bash
# Resolve a tag to its current digest
chainctl images tags resolve cgr.dev/chainguard/nginx:latest

# Resolve for all platforms (multi-arch)
chainctl images tags resolve cgr.dev/chainguard/nginx:latest --all
```

### `chainctl images repos`
Aliases: `repositories`, `repository`, `repo`. Image repository management.

| Command | Description |
|---------|-------------|
| `list` | List image repositories |
| `create` | Create an image repository |
| `delete` | Remove an image repository |
| `update` | Update image repositories |
| `build` | Manage Custom Assembly builds (subcommands: `apply`, `edit`, `list`, `logs`) |

#### `chainctl images repos create`
Create an image repository. Aliases: `make`, `mk`.

**Flags:**
- `--parent` — Name or ID of parent location
- `--description` — Description for the repo (max 255 characters)
- `--source` — Repository ID to sync from

**Required Capabilities:** `groups.list`, `repo.create`

#### `chainctl images repos list`
List image repositories. Aliases: `ls`.

**Flags:**
- `--parent` — Name or ID of parent location
- `--public` — List repos from the public Chainguard registry
- `--recursive` — Search recursively through all descendants
- `--repo` — Search for a specific repo by name

**Required Capabilities:** `groups.list`, `repo.list`

#### `chainctl images repos delete`
Remove an image repository. Aliases: `rm`.

**Flags:**
- `--parent` — Name or ID of parent location
- `--allow-missing` — Exit with status 0 if the repo does not exist

**Required Capabilities:** `repo.list`, `repo.delete`

#### `chainctl images repos update`
Update image repositories.

**Flags:**
- `--parent` — Name or ID of parent location
- `--name` — Updated name for the repo
- `--description` — Updated description
- `--bundles` — Comma-separated list of bundles to assign
- `--tier` — Catalog tier: `COMMERCIAL`, `APPLICATION`, `BASE`, `FIPS`, `AI`, `DEVTOOLS`
- `--expiration` — Sync expiration time (e.g. `1969-12-31`)

**Required Capabilities:** `groups.list`, `repo.update`, `repo.list`

### Custom Assembly (chainctl images repos build)

Custom Assembly lets you customize any entitled Chainguard image by adding packages, environment variables, OCI annotations, custom user accounts/groups, and custom certificates — without forking images or maintaining custom build pipelines. The customized image is built automatically.

**Required Capabilities:** `groups.list`, `repo.create`, `repo.update`, `manifest.create`, `tag.list`, `apk.list`, `build_report.list`

#### YAML Configuration Sections

Custom Assembly uses a YAML configuration manifest with these sections:

| Section | Description |
|---------|-------------|
| `contents.packages` | Additional packages to install (must be in Chainguard's package repo) |
| `environment` | Environment variables for the image (`CHAINGUARD_` prefix is reserved) |
| `annotations` | Custom OCI annotations (`dev.chainguard` and `org.opencontainers` prefixes are reserved) |
| `accounts` | Custom users/groups with UIDs/GIDs, home dirs, group memberships, and run-as user |
| `certificates` | Custom certs merged with the default bundle (Beta — requires enrollment) |

#### `chainctl images repos build edit`
Interactive editor to customize a Chainguard image. Opens your editor with the current config (or a template for new repos). Shows a diff for review before applying.

**Flags:**
- `--repo` — Name or ID of the repo to edit
- `--parent` — Name or ID of the parent location
- `-f, --file` — Pre-written config file (skips interactive editor)
- `--save-as` — Create a new repo with the edited config instead of modifying the existing one
- `--with-certificates` — Comma-separated list of certificate files to include (can be specified multiple times)

**Required Capabilities:** `groups.list`, `repo.create`, `repo.update`, `repo.list`

**Examples:**
```bash
# Edit interactively (prompts for repo selection)
chainctl images repos build edit

# Edit a specific repo
chainctl images repos build edit --repo=my-custom-python

# Edit and save as a new repo variant
chainctl images repos build edit --repo=my-custom-python --save-as=my-new-python

# Apply config from a file (non-interactive)
chainctl images repos build edit --repo=my-custom-python --file=config.yaml

# Add custom certificates
chainctl images repos build edit --repo=my-custom-python --with-certificates=ca1.pem --with-certificates=ca2.pem

# Combine file config with certificates
chainctl images repos build edit --file=config.yaml --with-certificates=internal-ca.pem
```

#### `chainctl images repos build apply`
Apply a YAML configuration file non-interactively. Ideal for CI/CD pipelines and GitOps workflows.

**Flags:**
- `--repo` — Name or ID of the repo
- `--parent` — Name or ID of the parent location
- `-f, --file` — Config file to apply
- `--save-as` — Create a new repo instead of updating existing
- `--with-certificates` — Certificate files to include (at least one of `--file` or `--with-certificates` is required)
- `-y, --yes` — Auto-confirm (for CI/CD)

**Required Capabilities:** `groups.list`, `repo.update`, `repo.list`

**Examples:**
```bash
# Apply config from a file
chainctl images repos build apply --repo=my-custom-python --file=config.yaml

# Apply and save as a new repo
chainctl images repos build apply --repo=my-custom-python --file=config.yaml --save-as=my-new-python

# CI/CD: apply with auto-confirm
chainctl images repos build apply --repo=my-custom-python --file=config.yaml --yes

# Apply with custom certificates
chainctl images repos build apply --repo=my-custom-python --file=config.yaml --with-certificates=ca1.pem --with-certificates=ca2.pem
```

#### `chainctl images repos build list`
List build reports. Aliases: `ls`.

**Flags:**
- `--parent` — Name or ID of parent location
- `--repo` — Search for a specific repo by name or ID
- `--recursive` — Search recursively through descendants

**Required Capabilities:** `groups.list`, `repo.list`, `tag.list`, `build_report.list`

#### `chainctl images repos build logs`
Get build logs for a specific build. Aliases: `log`.

**Flags:**
- `--repo` — Name or ID of the repo
- `--parent` — Name or ID of parent location
- `--build-id` — ID of the specific build

**Required Capabilities:** `groups.list`, `repo.list`, `tag.list`, `build_report.list`

**Examples:**
```bash
# Get logs for a repo (interactive build selection)
chainctl images repos build logs --repo=my-custom-python --parent my-org

# Get logs for a specific build
chainctl images repos build logs --repo=my-custom-python --build-id=abc123
```

### `chainctl images advisories list`
List security advisories for packages in an image. Aliases: `advisory`, `adv`.

**Positional arg:** `IMAGE_REF` (any OCI ref with SBOM attestations attached).

**Flags:**
- `--platform` — Platform to fetch SBOM for (default `linux/amd64`)
- `--status` — Filter advisories by status; repeatable or comma-separated. Values: `detected`, `pending-upstream`, `fixed`, `false-positive`, `not-affected`.

**Required Capabilities:** `advisories.list`

**Caveat:** only queries **APK packages** listed in the image's SBOM attestation. Images without SBOM attestations return no results.

### `chainctl images entitlements list`
List registry entitlements of an organization.

**Flags:**
- `--parent` — Org or folder to list entitlements for
- `-a, --all` — Include expired entitlements

**Required Capabilities:** `groups.list`, `registry.entitlements.list`

### `chainctl images helm values`
Generate relocation overrides for a Chainguard Helm chart.

**Flags:**
- `--registry` — Override the registry host for all image references
- `--org` — Override the organization prefix for image repositories

**Behavior:** if **neither** flag is set, output is **empty**. Intended for use with `helm install -f <(chainctl images helm values ...)`. Works for both `cgr.dev/<org>/charts/<name>` and iamguarded (Bitnami-fork) charts at `cgr.dev/<org>/iamguarded-charts/<name>` (requires a `global.org` chart value).

---

## events — Event Subscriptions

`chainctl events` (aliases: `event`)

### `chainctl events subscriptions`
Aliases: `subscription`, `subs`, `sub`; subcommand aliases: `list`/`ls`, `delete`/`rm`, `create`/`make`/`mk`.

| Command | Description |
|---------|-------------|
| `list` | List subscriptions (caps: `groups.list`, `subscriptions.list`). Flag: `--parent`. |
| `create` | Subscribe to events under an organization or folder |
| `delete` | Delete a subscription (caps: `subscriptions.delete`). Flag: `-y, --yes`. |

#### `chainctl events subscriptions create`
Subscribe to events. Aliases: `make`, `mk`.

**Usage:** `chainctl events subscriptions create [flags] <SINK_URL>` — the sink URL is a **positional argument** (an HTTPS webhook endpoint like `https://webhook.example/chainguard`, a Slack app webhook, an AWS Lambda URL, a Cloud Run endpoint, etc.). Not a flag.

**Flags:**
- `--parent` — Parent location name or ID for the subscription
- `-y, --yes` — Auto-confirm prompts

**Required Capabilities:** `groups.list`, `subscriptions.create`

**Filtering:** subscriptions fan out **all** events under the parent group. Filter on the `Ce-Type` CloudEvent header at your webhook — not on the subscription itself (no CEL on subscriptions).

**Webhook validation:** Chainguard signs deliveries with a JWT in the `Authorization` header. Validate:
- `iss = https://issuer.enforce.dev`
- `sub = webhook:<UIDP>`
- JWT digest vs content body

**Source IPs to allow-list:** `34.132.193.40`, `35.237.242.37`, `35.230.121.20`.

**Common event types (`Ce-Type`):**
- `dev.chainguard.registry.pull.v1` / `push.v1`
- `dev.chainguard.api.auth.registered.v1`
- `dev.chainguard.api.events.subscription.created.v1` / `.deleted.v1`
- `dev.chainguard.api.iam.{group,group_invite,identity,identity_providers,rolebindings,account_associations}.{created,updated,deleted}.v1`
- `dev.chainguard.api.platform.registry.{repo,tag}.{created,updated,deleted}.v1`

**Chainguard Notifications vs CloudEvents:** "Chainguard Notifications" (Console → Slack/email, sent by Customer Success for breaking changes / EOL / incidents) is a **separate** feature from CloudEvents subscriptions. Users often conflate them.

---

## packages — Package Management

`chainctl packages` (aliases: `package`, `pkg`, `pkgs`)

### `chainctl packages versions list`
List package version data from Chainguard repositories. Aliases: `ls`.

**Flags:**
- `--include-inactive` — Include packages within the EOL grace period end date
- `--show-active` — Show only active versions
- `--show-eol` — Show only EOL versions
- `--show-fips` — Show only FIPS versions

**Required Capabilities:** `version.list`

---

## libraries — Ecosystem Libraries

`chainctl libraries` (aliases: `libs`, `ecosystems`)

### `chainctl libraries verify`
Analyze artifacts to determine how much was built from source by Chainguard, using SBOM data, signatures, and artifact inspection. Aliases: `check`.

**Supports:** directories, archives, packages, container images (registry refs, local images, `docker-archive:` format). Package-manager caches auto-detected: npm (`_cacache/index-v5/`), pnpm (`v10/index/`, `v11/index/`), Yarn Classic (`yarn:` prefix).

**Flags:**
- `-d, --detailed` — Show detailed per-artifact results
- `--no-color` — Disable colored output
- `-o, --output` — Output format: `text`, `json`, `yaml`, `csv`
- `--verbose` — Enable verbose output
- `--parent` — Parent organization for authentication
- `--ecosystems-url` — URL for the Ecosystems Proxy (default `https://libraries.cgr.dev`)

**Output:** reports a **Verification Coverage percentage** per artifact (e.g. 100.00% = fully from Chainguard sources, 0% = none). Uses cosign-verified SLSA attestations signed against `issuer.enforce.dev`, with signature-based identification falling back to checksums.

**Fat/uber/shaded JARs always return 0%** — verify individual JARs from `~/.m2/repository` *before* assembly.

**Examples:**
```bash
# Analyze a JAR file
chainctl libraries verify myapp.jar

# Analyze multiple files
chainctl libraries verify build/libs/*.jar build/libs/*.war

# Analyze a Python virtual environment
chainctl libraries verify ./venv/

# Analyze a container image
chainctl libraries verify cgr.dev/chainguard/maven:latest

# JSON output with details
chainctl libraries verify -o json -d build/libs/*.jar
```

### `chainctl libraries entitlements`
Manage entitlements to language ecosystem libraries. Aliases: `entitlement`.

| Command | Description |
|---------|-------------|
| `create` | Create ecosystem library entitlements for an organization |
| `delete` | Delete an ecosystem library entitlement from an organization |
| `list` | List entitlements of an organization |

#### `chainctl libraries entitlements create`
Create ecosystem library entitlements. Aliases: `make`, `mk`.

**Flags:**
- `--ecosystems` — Language ecosystems to entitle: `JAVASCRIPT`, `JAVA`, `PYTHON` (comma-separated for multiple). Note `create` uses `--ecosystems` (plural); `delete` uses `--ecosystem` (singular).
- `--parent` — Name or ID of the org to create an entitlement for
- `--policy` — Policy to apply: `CHAINGUARD` (Chainguard-only) or `CHAINGUARD_AND_UPSTREAM` (Chainguard repo with upstream fallback; **documented only for JAVASCRIPT**)
- `--cooldown-days` — Days upstream package versions must wait before being served (meaningful only with `CHAINGUARD_AND_UPSTREAM`). Default 7. Valid range: **3–3650**.

**Required Capabilities:** `groups.list`, `libraries.entitlements.create`

**Ecosystem-specific caveats:**
- **No SLA on `libraries.cgr.dev`** — proxy through an artifact manager (Artifactory, Nexus) for production.
- **Chainguard checksums differ from upstream.** Lockfiles pinned to upstream hashes will fail strict-verification — always relock after adoption.
- **Java**: no snapshot versions, no source/Javadoc JARs, no distribution tarballs in most cases. Must purge `~/.m2/repository` AND `~/.gradle/caches/` and any `mavenLocal()` before first resolve. Env var idiom: `CHAINGUARD_JAVA_IDENTITY_ID` / `CHAINGUARD_JAVA_TOKEN`.
- **JavaScript**: malware scanning via OSV/OpenSSF Malicious Packages — MAL-flagged packages permanently blocked. **Artifactory redirect caching must be disabled** — `libraries.cgr.dev` uses Cloudflare R2 with 302 redirects; Artifactory's default may cache the redirect URL instead of blob content. For private scoped packages, add `replace-registry-host=never` to `.npmrc`. Pull-token default TTL 30 days.
- **Python**: Chainguard ships **manylinux** native binaries only (glibc ≥ 2.28, x86_64/aarch64). Dev workflow on **Windows/macOS falls back to PyPI** for native-extension packages — configure a PyPI fallback or use WSL2/Docker. Three separate indexes: `/python/` (baseline), `/python-remediated/` (adds `+cgr.N` local-version suffix with CVE remediations — unique to Python), and CUDA-specific `/cu126/`, `/cu128/`, `/cu129/` (not dependency-complete — configure NVIDIA's or PyPI for toolkit components). For uv, set `index-strategy = unsafe-best-match` when using the remediated index. Short-lived credentials via `pip install keyrings-chainguard-libraries`. Env vars: `CHAINGUARD_PYTHON_IDENTITY_ID` / `CHAINGUARD_PYTHON_TOKEN`.
- **Go**: not supported by Chainguard Libraries — no `--repository=go` for pull tokens.

**Examples:**
```bash
# Set up access to all ecosystems
chainctl libraries entitlements create --ecosystems=JAVASCRIPT,JAVA,PYTHON

# Set up access to just JavaScript
chainctl libraries entitlements create --ecosystems=JAVASCRIPT

# Set up JavaScript with Chainguard and upstream enabled
chainctl libraries entitlements create --ecosystems=JAVASCRIPT --policy=CHAINGUARD_AND_UPSTREAM

# Chainguard-only policy (disables upstream fallback)
chainctl libraries entitlements create --ecosystems=JAVASCRIPT --policy=CHAINGUARD

# Set up access with organization specified
chainctl libraries entitlements create --ecosystems=JAVASCRIPT --parent=orgname.com
```

#### `chainctl libraries entitlements list`
List entitlements of an organization. Aliases: `ls`.

**Flags:**
- `--parent` — Name or ID of the org to list entitlements for

**Required Capabilities:** `groups.list`, `libraries.entitlements.list`

#### `chainctl libraries entitlements delete`
Delete an ecosystem library entitlement. Aliases: `rm`.

**Flags:**
- `--ecosystem` — Language ecosystem to remove the entitlement for
- `--parent` — Name or ID of the org to delete an entitlement from

**Required Capabilities:** `groups.list`, `libraries.entitlements.list`, `libraries.entitlements.delete`

---

## agent — Agent Commands (The Guardener)

`chainctl agent`

The Guardener is an AI-powered migration agent that converts Dockerfiles to use Chainguard Containers. It iteratively converts instructions, builds images, compares results using syft, and fixes issues until the Dockerfile works as expected. The AI runs server-side; Docker builds and file access remain local.

**Note:** The Guardener is currently in beta. Your organization must join the waitlist at the Guardener landing page.

**Prerequisites:**
- `chainctl` installed and authenticated
- Docker installed and running locally
- `repo.create` capability in your Chainguard organization
- Dockerfile and build context present on the same machine

### `chainctl agent accept-terms`
Accept required legal terms for a group.

**Flags:**
- `--group` — UIDP of the group to accept terms for

### `chainctl agent dockerfile`
AI-powered Dockerfile migration commands.

| Command | Description |
|---------|-------------|
| `build` | Migrate a Dockerfile to use Chainguard wolfi-base images |
| `optimize` | Optimize an already-migrated wolfi-based Dockerfile |
| `upgrade` | Upgrade package versions in a Dockerfile |
| `validate` | Validate a migrated Dockerfile |

**Guardener caveats to set expectations:**
- **Migration target** is `cgr.dev/chainguard/wolfi-base:latest` only. The `multi-stage` optimizer can switch the runtime stage to distroless `-dev` / runtime variants.
- **Time to run**: 5–30+ minutes per migration. Bump Bash timeout to 1800000 ms.
- **Non-determinism**: AI-based — expect slightly different results across runs.
- **`--resume` only restores local state**, not the server-side session. If the connection drops, the server-side agent and in-flight conversation are lost; resume replays local progress only.
- **No server-side builds**: Docker must run locally. Fully-managed headless mode is planned but not available.
- Tested ecosystems: Python, Go, Node.js, Java, Spring Boot (including UBI-based sources), multi-stage Argo CD builds.
- **`accept-terms`** is required on first use; the server prompts for it when missing. Run manually with `chainctl agent accept-terms --group <UIDP>` if you want to pre-accept.
- **Output streams**: `build` prints its report to stdout; `optimize` and `upgrade` print reports to stderr; `validate` prints its report to stdout.
- **`--help` for `chainctl agent`** only shows `accept-terms`, but `dockerfile` exists and works.

#### `chainctl agent dockerfile build`
Migrate an existing Dockerfile to use Chainguard wolfi-base images. The agent iteratively builds and tests the migrated Dockerfile. Output defaults to `.migrated`.

**Flags:**
- `-f, --dockerfile` — Path to the Dockerfile
- `-t, --tag` — Image tag for build testing
- `-o, --output` — Output path for migrated Dockerfile (use `-` for stdout)
- `--group` — Chainguard group ID for authorization
- `--non-interactive` — Run without user prompts (for CI)
- `--resume` — Resume a previous migration session
- `--build-arg` — Build arguments (`key=value`)
- `--platform` — Set target platform for build
- `--target` — Set target build stage
- `--no-cache` — Do not use cache when building
- `--log-file` — Write detailed log output to a file
- `--no-color` — Disable colored output in reports
- Plus all standard Docker build flags (`--add-host`, `--cache-from`, `--secret`, `--ssh`, etc.)

**Examples:**
```bash
# Migrate the default Dockerfile in the current directory
chainctl agent dockerfile build

# Migrate a specific Dockerfile with build testing
chainctl agent dockerfile build -f path/to/Dockerfile -t myimage:test --group <group-id>

# Migrate with build arguments
chainctl agent dockerfile build -f Dockerfile -t myapp:chainguard --build-arg VERSION=1.0 --group <group-id>

# Non-interactive migration (CI/CD)
chainctl agent dockerfile build -f Dockerfile --non-interactive --group <group-id>

# Write migrated Dockerfile to stdout
chainctl agent dockerfile build -f Dockerfile --non-interactive -o -

# Resume a previous migration session
chainctl agent dockerfile build -f Dockerfile --resume --group <group-id>
```

#### `chainctl agent dockerfile optimize`
Optimize an existing wolfi-based Dockerfile for size, security, and best practices. Prints an optimization report to stderr. Output defaults to `.optimized`.

**Flags:**
- `-f, --dockerfile` — Path to the Dockerfile
- `-o, --output` — Output path (use `-` for stdout)
- `--group` — Chainguard group ID for authorization
- `--optimizers` — Comma-separated list of specific optimizers to run

**Available Optimizers:**
- `cache` — Reorder instructions for better layer caching (faster builds)
- `cleanup` — Remove duplicate and redundant instructions
- `layers` — Combine RUN commands and merge package installs (smaller image)
- `security` — Add `--no-cache` to apk, flag secrets, suggest non-root USER
- `multi-stage` — Transform into multi-stage build using Chainguard runtime images
- `native-packages` — Replace curl/bash installs with native apk packages

**Examples:**
```bash
# Optimize the default Dockerfile
chainctl agent dockerfile optimize

# Optimize with specific optimizers
chainctl agent dockerfile optimize -f Dockerfile --optimizers=cache,security --group <group-id>

# Write optimized Dockerfile to a specific path
chainctl agent dockerfile optimize -f Dockerfile -o Dockerfile.new
```

#### `chainctl agent dockerfile upgrade`
Upgrade package versions to their latest available versions in the Chainguard package repository. Prints an upgrade report to stderr. Output defaults to `.upgraded`.

**Flags:**
- `-f, --dockerfile` — Path to the Dockerfile
- `-o, --output` — Output path (use `-` for stdout)
- `--group` — Chainguard group ID for authorization
- `--dry-run` — Show upgrade plan without applying changes
- `--non-interactive` — Run without user prompts

**Examples:**
```bash
# Upgrade packages in the default Dockerfile
chainctl agent dockerfile upgrade

# Preview what would be upgraded (dry run)
chainctl agent dockerfile upgrade -f Dockerfile --dry-run --group <group-id>

# Upgrade non-interactively and write to stdout
chainctl agent dockerfile upgrade -f Dockerfile --non-interactive -o -
```

#### `chainctl agent dockerfile validate`
Validate a migrated Dockerfile for correctness and best practices. Checks that packages exist, base images are valid, and the Dockerfile follows recommended patterns.

**Flags:**
- `-f, --dockerfile` — Path to the Dockerfile
- `--group` — Chainguard group ID for authorization
- `--level` — Validation level: `0` = standard, `1` = strict (enforces additional best practices)

**Examples:**
```bash
# Validate the default Dockerfile
chainctl agent dockerfile validate

# Validate with strict checks
chainctl agent dockerfile validate -f Dockerfile --level 1

# Validate a migrated Dockerfile
chainctl agent dockerfile validate -f Dockerfile.migrated --group <group-id>
```

---

## completion — Shell Autocompletion

`chainctl completion`

Generate autocompletion scripts for your shell.

| Command | Description |
|---------|-------------|
| `bash` | Generate bash autocompletion script |
| `fish` | Generate fish autocompletion script |
| `powershell` | Generate PowerShell autocompletion script |
| `zsh` | Generate zsh autocompletion script |

**Examples:**
```bash
# Generate zsh completions
chainctl completion zsh > "${fpath[1]}/_chainctl"

# Generate bash completions
chainctl completion bash > /etc/bash_completion.d/chainctl
```

---

## update & version

- `chainctl update` — Update chainctl to the latest version (alias: `chainctl upgrade`)
  - `--force` — Skip the version check and update regardless of the current version
  - `-y, --yes` — Auto-confirm
  - Auto-verifies the cosign signature on the new binary and refuses to replace on verification failure.
  - Leaves `~/.cache/chainctl/chainctl.bak` after updating — remove if disk hygiene matters (especially on CI runners).
- `chainctl version` — Print the current version

---

## Installation — beyond macOS Homebrew

```bash
# macOS (Homebrew)
brew install chainguard-dev/tap/chainctl   # requires Xcode Command Line Tools

# Linux / macOS direct download
curl -o chainctl "https://dl.enforce.dev/chainctl/latest/chainctl_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/aarch64/arm64/')"
chmod +x chainctl && sudo mv chainctl /usr/local/bin/

# Windows
curl -o chainctl.exe https://dl.enforce.dev/chainctl/latest/chainctl_windows_x86_64.exe
# Invoke as `.\chainctl`. Some commands are less tested on Windows.
# For configure-docker: create a symlink `docker-credential-cgr.exe` alongside chainctl.exe.
```

**Cosign verification recipe (regulated customers should always do this):**
```bash
cosign verify-blob \
  --certificate chainctl.crt \
  --signature chainctl.sig \
  --certificate-identity-regexp "^https://github.com/chainguard-dev/mono/.github/workflows/.*" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  chainctl
```

**Shell completions:**
```bash
chainctl completion zsh > "${fpath[1]}/_chainctl"
chainctl completion bash > /etc/bash_completion.d/chainctl
chainctl completion fish > ~/.config/fish/completions/chainctl.fish
```

---

## Network requirements

**Hosts to allow-list** (most have 60-second DNS TTL — re-resolve often):

| Host | Purpose |
|------|---------|
| `cgr.dev` | Container registry |
| `console.chainguard.dev` | Web console |
| `data.chainguard.dev` | Console API |
| `console-api.enforce.dev` | Registry / Platform API |
| `enforce.dev`, `dl.enforce.dev` | Binary downloads |
| `issuer.enforce.dev` | STS token service |
| `auth.chainguard.dev` | Headless device-code activation |
| `apk.cgr.dev`, `virtualapk.cgr.dev`, `packages.cgr.dev` | Private APK repos |
| `packages.wolfi.dev` | Free-image APK repo |
| `libraries.cgr.dev` | Chainguard Libraries proxy |
| `9236a389bd48b984df91adc1bc924620.r2.cloudflarestorage.com` | Blob storage (R2) for `*.cgr.dev` |
| `support.chainguard.dev` | Support portal |

**Zscaler gotcha (common support issue):** Chainguard endpoints require encrypted **HTTP/2**. Zscaler downgrades non-browser HTTP/2 to HTTP/1.1 by default. Fix by (1) opening a Zscaler TAC ticket to enable HTTP/2 at the backend, (2) toggling HTTP/2 in SSL Inspection policies, (3) enabling the "API/CLI HTTP/2" advanced setting. Workaround: add the hosts above to `no_proxy`.

**TLS requirements:** minimum TLSv1.3, or TLSv1.2 with RFC 7627 (Extended Master Secret). Some legacy FIPS 140-2 middleware will fail. FIPS containers require TLS 1.3.

---

## Image lifecycle (EOL / grace period)

- Chainguard publishes **EOL grace periods up to 6 months** after upstream EOL — not guaranteed; the grace period ends **immediately if a build fails due to dependency conflict**.
- `chainctl packages versions list --include-inactive` shows packages in the grace window.
- Listing EOL tags programmatically:
  ```bash
  REPO_UIDP=$(chainctl images repos list --parent my-org -o id | grep node | head -1)
  curl -s -H "Authorization: Bearer $(chainctl auth token)" \
    "https://console-api.enforce.dev/registry/v1/eoltags?uidp.childrenOf=${REPO_UIDP}" | jq .
  ```
- API returns: `tagStatus` ∈ {`TAG_ACTIVE`, `TAG_IN_GRACE`, `TAG_INACTIVE`}; `graceStatus` ∈ {`GRACE_ACTIVE`, `GRACE_ELIGIBLE`, `GRACE_NOT_ELIGIBLE`}.

---

## Compliance — FIPS / STIG / FedRAMP

- **Ask the user about compliance level** early — it affects image variant selection and TLS posture.
- FIPS image naming: `-fips` suffix (e.g., `python-fips`, `nginx-fips`, `jdk-fips`, `jre-fips`, `go-fips`, `glibc-openssl-fips`).
- FIPS images use **kernel-independent** Jitterentropy userspace entropy — run on any recent Linux kernel (including dev laptops). Caveats: some CNI plugins, LUKS2, StrongSwan still need FIPS-mode kernel.
- Verify FIPS-hardened config in SBOM: look for `openssl-config-fipshardened` and `libcrypto3`.
- STIG scanning: use the `openscap` Chainguard image with `ssg-chainguard-gpos-ds.xml` datastream.

---

## Attestations & signatures — cosign workflow

chainctl does not itself verify signatures; use `cosign` alongside. Every Chainguard image has:

| Predicate type | Purpose |
|----------------|---------|
| `https://slsa.dev/provenance/v1` | SLSA 1.0 build provenance |
| `https://spdx.dev/Document` | SPDX SBOM |
| `https://cyclonedx.org/bom` | CycloneDX SBOM (customer-only, builds ≥ 2026-01-29) |
| `https://apko.dev/image-configuration` | apko build configuration |
| `https://chainguard.dev/end-of-life` | EOL metadata |
| `https://chainguard.dev/helm-values/v1` | Helm values attestation |
| `https://chainguard.dev/attestation/chart-lock/v1` | Helm chart-lock |
| `https://chainguard.dev/attestation/syft/v1` | Syft SBOM |

Example:
```bash
cosign verify-attestation --type https://slsa.dev/provenance/v1 cgr.dev/$ORG/nginx:1.25
cosign download attestation --predicate-type https://spdx.dev/Document cgr.dev/$ORG/nginx:1.25 | jq -r .payload | base64 -d | jq .
```

Referrer tags (`sha256-<digest>.sig|sbom|att`) are exposed via `chainctl images list --show-referrers`.

---

## Troubleshooting

1. **Verbose output for bug reports:** `chainctl --v 5 <cmd> 2>&1 > run.log`. Also attach `chainctl version` and `chainctl config view` (scrub secrets). Alternatively use `--log-level debug`.
2. **Force re-authentication:** delete the token cache — `rm -rf ~/.cache/chainguard/<audience>/` (Linux) or `~/Library/Caches/chainguard/<audience>/` (macOS). The audience's `/` is replaced with `-` on disk.
3. **401s in long-running CI**: check whether the pull token has expired (default 30 days). Tokens rotate via refresh-token — if refresh failed, re-run `configure-docker --pull-token --save`.
4. **Stuck Guardener migration**: use `--log-file <path>` to capture detailed logs; extend Bash timeout to 1800000 ms or longer.
5. **`libraries verify` returns 0% on fat JARs**: expected — verify individual JARs from `~/.m2/repository` before assembly.
6. **Custom Assembly build timeout (~1 hour)**: normal builds complete in <20 min. Use `chainctl images repos build logs --repo=<name> --build-id=<id>` to read server-side logs.
7. **npm 404 on a brand-new upstream version**: expected during the 7-day cooldown on `CHAINGUARD_AND_UPSTREAM`; retry after the window, or lower `--cooldown-days` on the entitlement.
8. **Stale `chainctl update` cache**: `rm -f ~/.cache/chainctl/chainctl.bak`.

---

## Common Workflows

### First-time setup
```bash
# Install (macOS)
brew install chainguard-dev/tap/chainctl

# Login
chainctl auth login

# Check status
chainctl auth status

# View config
chainctl config view
```

### Docker image pulling setup
```bash
# Configure Docker credential helper
chainctl auth configure-docker

# Or with a pull token for CI/CD
chainctl auth configure-docker --pull-token --save --parent my-org --ttl 720h
```

### npm / JavaScript library setup
```bash
# Configure npm using your current session
chainctl auth configure-npm

# Configure npm with a pull token for CI/CD
chainctl auth configure-npm --pull-token --parent=my-org --ttl=24h
```

### Explore available images
```bash
# List public images
chainctl images list --public

# List images in your org
chainctl images list --parent my-org

# Check image history
chainctl images history nginx:latest --parent my-org

# View changelog
chainctl images changelog cgr.dev/chainguard/nginx:latest

# Check security advisories
chainctl images advisories list cgr.dev/chainguard/nginx:latest

# Diff two images
chainctl images diff cgr.dev/chainguard/nginx:latest cgr.dev/chainguard/nginx:1.25
```

### IAM management
```bash
# List orgs
chainctl iam organizations list

# List folders in an org
chainctl iam folders list --parent my-org

# List identities
chainctl iam identities list --parent my-org

# List roles
chainctl iam roles list --parent my-org

# List role-bindings
chainctl iam role-bindings list --parent my-org

# Create an invite
chainctl iam invites create --parent my-org

# Check cloud account associations
chainctl iam account-associations describe --parent my-org
```

### Custom Assembly workflow
```bash
# Edit an image interactively (opens editor with YAML config)
chainctl images repos build edit --repo=my-custom-python --parent my-org

# Create a variant of an image
chainctl images repos build edit --repo=python --save-as=my-custom-python --parent my-org

# Apply config from file (CI/CD)
chainctl images repos build apply --repo=my-custom-python --file=config.yaml --yes

# Apply with custom certificates
chainctl images repos build apply --repo=my-app --file=config.yaml --with-certificates=internal-ca.pem --yes

# List builds for a repo
chainctl images repos build list --repo=my-custom-python --parent my-org

# Check build logs
chainctl images repos build logs --repo=my-custom-python --parent my-org
```

### Dockerfile migration (The Guardener)
```bash
# Migrate a Dockerfile to use Chainguard images
chainctl agent dockerfile build -f Dockerfile -t myapp:chainguard --group <group-id>

# Optimize the migrated Dockerfile
chainctl agent dockerfile optimize -f Dockerfile.migrated --group <group-id>

# Upgrade packages to latest versions
chainctl agent dockerfile upgrade -f Dockerfile --group <group-id>

# Validate the result
chainctl agent dockerfile validate -f Dockerfile.migrated --group <group-id>

# Full CI/CD pipeline (non-interactive)
chainctl agent dockerfile build -f Dockerfile --non-interactive --group <group-id>
```

### Library entitlements management
```bash
# Set up access to all ecosystems
chainctl libraries entitlements create --ecosystems=JAVASCRIPT,JAVA,PYTHON --parent=my-org

# Set up JavaScript with upstream fallback enabled
chainctl libraries entitlements create --ecosystems=JAVASCRIPT --policy=CHAINGUARD_AND_UPSTREAM

# List entitlements
chainctl libraries entitlements list --parent=my-org

# Remove an entitlement
chainctl libraries entitlements delete --ecosystem=JAVASCRIPT --parent=my-org
```

### Library verification
```bash
# Verify a container image uses Chainguard libraries
chainctl libraries verify cgr.dev/chainguard/maven:latest

# Verify local artifacts
chainctl libraries verify ./build/libs/*.jar -d -o json

# Verify an npm cache
chainctl libraries verify "$(npm config get cache)"

# Verify a pnpm store
chainctl libraries verify "$(pnpm store path)"

# Verify a Yarn Classic cache
chainctl libraries verify yarn:
```

### Verify a Chainguard image with cosign
```bash
# Verify SLSA provenance
cosign verify-attestation --type https://slsa.dev/provenance/v1 cgr.dev/$ORG/nginx:1.25

# Download and inspect the SPDX SBOM
cosign download attestation --predicate-type https://spdx.dev/Document cgr.dev/$ORG/nginx:1.25 \
  | jq -r .payload | base64 -d | jq .
```

### Pin an image by digest for reproducibility
```bash
# Resolve all tags for a digest
chainctl images tags resolve cgr.dev/$ORG/nginx:1.25 --all

# Or read history for audit
chainctl images history nginx:1.25 --parent $ORG

# Then in Dockerfile: FROM cgr.dev/$ORG/nginx@sha256:...
```

### CVE remediation via diff
```bash
# See what vulnerabilities were resolved between two tags
chainctl images diff cgr.dev/$ORG/go:1.21.2 cgr.dev/$ORG/go:1.21.5 -o json \
  | jq '.vulnerabilities'
```

### Revoke a pull token
```bash
# Pull tokens ARE identities; to revoke, delete the identity
chainctl iam identities list --parent $ORG
chainctl iam identity delete <UUID>
```

### Helm chart relocation
```bash
helm install r oci://cgr.dev/$ORG/charts/foo \
  -f <(chainctl images helm values cgr.dev/$ORG/charts/foo:latest \
         --registry myregistry.example.com --org other-org)
```

### Set up an event subscription
```bash
# Sink URL is POSITIONAL
chainctl events subscriptions create --parent $ORG https://webhook.example/chainguard
chainctl events subscriptions list --parent $ORG
chainctl events subscriptions delete <sub-id>
```
Filter events at your webhook by the `Ce-Type` header. Allow-list source IPs `34.132.193.40`, `35.237.242.37`, `35.230.121.20`.

### Use a private APK repo from a Custom Assembly `-dev` container
```bash
docker run -e "HTTP_AUTH=basic:apk.cgr.dev:user:$(chainctl auth token --audience apk.cgr.dev)" \
  cgr.dev/$ORG/my-custom:latest-dev
# Then inside: apk update && apk add <pkg>
```

---

## Tips

1. **Output formats** — Use `-o json` for scripting, `-o table` for readability, `-o tree` for hierarchy, `-o wide` for all fields, `-o id` for just IDs.
2. **Aliases** — Many commands have short aliases: `orgs`, `img`, `pkg`, `libs`, etc.
3. **Get help** — Append `--help` to any command for detailed flag info.
4. **Config file** — Set `CHAINCTL_CONFIG` env var or use `--config` to point to a specific config.
5. **Headless/CI** — Use `--headless` for non-interactive login, `--identity-token` for CI/CD pipelines.
6. **Pull tokens** — For CI/CD image pulling, create pull tokens with appropriate TTL and save to Docker config.
