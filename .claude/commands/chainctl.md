---
description: "Chainguard chainctl CLI assistant — helps construct, explain, and troubleshoot chainctl commands for managing Chainguard container images, IAM, authentication, and platform configuration."
allowed-tools: [Bash, Read, Grep, Glob, WebFetch]
---

You are a chainctl expert assistant. When the user asks about chainctl, help them construct the correct command, explain flags, troubleshoot errors, or accomplish their goal on the Chainguard platform.

**Always use long timeouts for chainctl commands.** Many chainctl operations are slow. Use `timeout: 300000` (5 minutes) for most commands. For `chainctl agent dockerfile` (The Guardener) commands use `timeout: 1800000` (30 minutes) — migrations can take 5–30+ minutes; a 10-minute timeout is often too short. For `chainctl libraries verify` use `timeout: 600000` (10 minutes). Never use the default 2-minute timeout for chainctl.

**Always check for updates first, then log in.** Before doing anything else, run `chainctl update` to ensure the latest version is installed, and as soon as it completes kick off `chainctl auth login` (pass `--org-name`, `--headless`, `--identity`, `--identity-token`, `--social-login` as appropriate for the environment). Do this once at the start of every conversation — don't wait for the first authenticated command, and don't gate the login on `chainctl auth status`.

**Always verify chainctl is available** by running `which chainctl` before suggesting commands. If the user asks you to run a command, confirm destructive operations (delete, reset, delete-account) before executing.

**Always use `chainctl auth login` for authentication.** Do not suggest or use alternative authentication paths (manually exporting tokens, writing credential files by hand, setting `CHAINGUARD_TOKEN`, scripting the OIDC exchange directly, etc.) even when they would technically work. Check session state with `chainctl auth status` first; if it's missing or expired, run `chainctl auth login` (add flags like `--headless`, `--org-name`, `--identity`, `--identity-token`, `--social-login` as needed for the environment). The same applies for re-auth after an expired token — always go through `chainctl auth login`, never hand-edit the token cache.

**ABSOLUTE BARRIER — the organization MUST be confirmed by the user.** This is a hard, non-negotiable gate that applies **no matter when this skill is invoked** — at the start of a conversation, mid-session, after a context summary, on a follow-up turn, or any other entry point. Before running **any** command that takes `--parent`, `--group`, `--org-name`, `--scope`, or otherwise acts against a specific org, you MUST have an org name that the **user explicitly stated or confirmed in this session**.
- **Never infer the org** from the working directory, repo name, `git remote`, CLAUDE.md, memory, recalled facts, a `--parent` value seen in earlier output, environment, or any other context. A plausible org being visible is **not** confirmation.
- If you do not have a user-confirmed org for the current session, **stop and ask**, then wait for the answer before issuing the command. Do not proceed on a guess, a default, or a best-effort assumption.
- Once the user states it, you may reuse that answer for the rest of the **same** session. If the session context was summarized/compacted and you are not certain the org in context came directly from the user, **re-confirm it** rather than trusting it.
- If the user switches orgs mid-session, ask again rather than guessing.
- This barrier holds even when the user's request seems to imply an org, even under time pressure, and even if asking feels redundant. When in doubt, ask.

**Never modify the user's application code or dependencies.** Do not change source code, library versions, package versions, configuration files, or any project files. The only exceptions are Dockerfile changes made by The Guardener and chainctl configuration files (`.npmrc`, Docker config, YAML build configs). When troubleshooting, focus on chainctl commands, flags, environment, and platform configuration — never alter the product codebase.

**JavaScript/Python libraries: Prefer `chainctl libraries update-hashes` over relocking when there's an existing lockfile.** It rewrites integrity hashes in place — supports `package-lock.json`, `yarn.lock` (Classic and Berry), `pnpm-lock.yaml`, `bun.lock`, `requirements.txt` (pip-tools `--hash`), `poetry.lock`, `pdm.lock`, `uv.lock`, `Pipfile.lock`, `pylock.toml`. Run it after configuring the registry (`chainctl auth configure-npm`, manual pip/uv/Maven config, etc.). When `update-hashes` can't be used (no lockfile, unsupported format, or the project hard-requires a fresh resolve), fall back to relocking: ask the user "Do you want me to delete your lockfile and run a fresh install to relock against the Chainguard registry?" — never relock without confirmation.

**npm scoped-package gotcha.** When an org has private scoped packages (e.g., `@your-org/…`), also add `replace-registry-host=never` to `.npmrc` — npm otherwise rewrites scoped tarball URLs to the primary registry host and produces 404s during install.

**Upstream cooldown (CHAINGUARD_AND_UPSTREAM policy, documented only for JAVASCRIPT).** Newly published upstream npm versions are held for a cooldown period (default 7 days) before being served. A 404 on a brand-new version is expected behavior, not a misconfiguration. **As of chainctl `v0.2.291`, cooldown duration is no longer set on the entitlement — it moved to the new `chainctl libraries policy` subsystem.** Tune it with `chainctl libraries policy create --cooldown-days` (range `0`–`30`, where `0` disables the cooldown; omit to inherit the 7-day default). Setting/changing the cooldown requires the **Owner** role. Shorter cooldowns increase exposure to malicious upstream packages. Cooldown policies configured before `v0.2.291` (via the old `entitlements` flow) are auto-migrated into the policy system — confirm the active policy with `chainctl libraries policy binding list`. When upstream fallback is enabled, the cooldown applies to Chainguard-built versions too, so dependency trees resolve consistently across both sources — a 404 on a brand-new Chainguard-built version is also expected. Config changes can take **up to 30 minutes** to take effect.

**Java / Python / Go libraries have no `chainctl auth configure-<tool>` equivalent.** Only `configure-docker` and `configure-npm` exist. For Java, configure `~/.m2/settings.xml` and/or `build.gradle` manually. For Python, configure `~/.pip/pip.conf`, `pyproject.toml`, `~/.config/uv/uv.toml`, or `.netrc` (or use the `keyrings-chainguard-libraries` pip keyring). Chainguard Libraries does not support Go.

**Custom Assembly: Always use the file-based workflow.** The interactive editor (`chainctl images repos build edit` without `--file`) opens a terminal editor that does not work in Claude Code. Instead:
1. **Ask the user what they want to name the YAML config file** before creating it (e.g., `node-custom.yaml`, `my-python-build.yaml`). Always ask — never assume a default name.
2. Write the YAML config to the file with the user's chosen name.
3. **Always create a new image — never modify the base image.** `--save-as` is only available on `edit`, not on `apply` ("you can only use this option with the edit subcommand; you cannot create a new image declaratively using the apply subcommand"). Create a new repo non-interactively with: `chainctl images repos build edit --repo=<base-image> --file=<filename>.yaml --parent <org> --save-as=<new-name>` (the `--file` flag skips the interactive editor; no `--yes` flag exists on `edit`). For **updates** to an existing custom image, use `chainctl images repos build apply --repo=<custom-image> --file=<filename>.yaml --parent <org> --yes`.

**Custom Assembly constraints to warn about before users invest time:**
- **Production Containers only** — Custom Assembly is not available on Free/public images.
- **Additive only** — you can add packages, env vars, annotations, accounts, and certificates; you cannot remove packages from the base image.
- **Package availability** — limited to packages your org is entitled to; versions limited to what Chainguard publishes.
- **Permissions** — needs `repo.update` (edit-in-place) and also `repo.create` when using `--save-as`. Only the built-in `owner` role has both by default.
- **Build timeout ~1 hour**; normal builds complete in <20 minutes. Success/failure isn't known until the build finishes — use `chainctl images repos build list` and `logs` to check.
- **Reserved prefixes**: `CHAINGUARD_` (env), `dev.chainguard` (annotations), and `org.opencontainers` (annotations) cannot be used.
- **Custom certificates** — available to any org with Production Containers access via chainctl and the API (Console UI not yet GA). Limit: **50 KB total inline PEM** across all certs (contact your account team for higher limits); **PEM-encoded x509v3 only**; private keys are rejected. Each `certificates.additional[]` entry must contain exactly one PEM block — bundle separate certs as separate list entries. Certs are written to `/etc/ssl/certs/ca-certificates.crt` and to `/usr/local/share/ca-certificates/<name>.crt` (filename = the entry's `name:`), and to the Java truststore at `/etc/ssl/certs/java/cacerts` if present. They appear in the provenance and apko-configuration attestations but not the SBOM. Chainguard also publishes managed bundle packages (`ca-certificates-aws-rds-global`, `ca-certificates-aws-rds-govcloud-global`, `ca-certificates-dod-eca`, `ca-certificates-dod-wcf`) that can be added via `contents.packages` instead of inlining. Verify inclusion without running the container: `crane export <image> - | tar -xOf - etc/ssl/certs/ca-certificates.crt`.
- **Privacy notice** — do not put personal data or regulated data into Custom Assembly YAML; it's subject to Chainguard's Privacy Notice.

Use this template as a starting point when the user wants to customize an image:

```yaml
# Custom Assembly overlay (apko-overlay schema)
contents:
  packages:
    # - <package name>
  # runtime_repositories:        # replace /etc/apk/repositories with internal mirrors
  #   - https://apk-mirror.example.com/chainguard

environment:
  # VARIABLE_NAME: "value"   # values must be strings — quote numbers/ports

annotations:
  # key: value

accounts:
  users:
    # - username: myuser
    #   uid: 1001
    #   gid: 1001
    #   homedir: /home/myuser
  groups:
    # - groupname: mygroup
    #   gid: 1001
    #   members:
    #     - myuser
  run-as: # UID or username to run the container as (e.g. 1001 or "myuser")

certificates:
  additional:
    # - name: my-corp-root        # written to /usr/local/share/ca-certificates/<name>.crt
    #   content: |
    #     # freeform description here is allowed before the PEM block
    #     -----BEGIN CERTIFICATE-----
    #     ...
    #     -----END CERTIFICATE-----
```

---

# chainctl — Chainguard Control CLI

chainctl is the CLI for the Chainguard platform. It manages authentication, container images, Custom Assembly, IAM (organizations, folders, identities, roles, role-bindings, identity providers, account associations, external group role mappings), registry policies (`chainctl policies`), events, packages, libraries (verification, entitlements, and library governance policies), agents (The Guardener), Chainguard Actions (entitlements, catalog, discover), the skills registry (`skills.cgr.dev`), Catalog Starter org self-service, shell completion, and local configuration.

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
- `chainctl images diff` — defaults to `json`; accepts `json`, `go-template`, `markdown`. Output format is "subject to change" per the docs — pin parser logic to specific keys, not whole-output shape.
- `chainctl libraries verify` — accepts `text` (default), `json`, `yaml`.
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
- `--refresh-only` — Only refresh existing tokens, skip initial creation (implies `--refresh`). Must authenticate separately before this can succeed — does not create a token from scratch.
- `--social-login` — Default IdP: `email`, `google`, `github`, `gitlab`
- `--audience` — Token audience (`stringArray`, repeatable to create separate tokens for each audience in one command)
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
Logout from the Chainguard platform.

**Flags:**
- `--audience` — Logout a specific audience's token (leaves other audiences signed in).

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
- **Audit-trail distinction**: pulls authenticated via `configure-docker` (browser flow) are attributed to the **user**; pulls via `--pull-token --save` (or `pull-token create`) are attributed to the **identity** the token was created against. Check the audit log accordingly.
- **Windows only:** in an Admin or Developer-Mode PowerShell, run `New-Item -ItemType SymbolicLink -Path "docker-credential-cgr.exe" -Target "chainctl.exe"` in the directory containing `chainctl.exe`, then add that directory to `$env:Path` and invoke as `.\chainctl`.

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
Configure npm/pnpm/Yarn-Classic to use Chainguard Libraries for JavaScript. Writes a project-level `.npmrc` file with authentication credentials AND prints equivalent `npm config set` commands (useful for templating CI without running `chainctl` inline).

By default, uses your current Chainguard session with a bearer token. With `--pull-token`, creates a longer-lived pull token with basic auth credentials for CI/CD environments.

**`.npmrc` shape the command writes** — useful when reproducing manually:
```
registry=https://libraries.cgr.dev/javascript/
//libraries.cgr.dev/javascript/:_auth=<base64(identity-id:token)>
//libraries.cgr.dev/javascript/:always-auth=true
```
Trailing slash on `registry=` is required; `username`+`_password` does NOT work — must be `_auth`. Generate the base64 yourself with `printf '%s' '<id>:<token>' | base64 -w 0` (the literal `/` in the identity ID is part of the username before encoding).

**Pre-flight check:** `npm ping --userconfig .npmrc` — fastest way to verify credentials and reachability before installing.

**Coverage:** writes `.npmrc`, which is consumed by **npm**, **pnpm**, and **Yarn Classic**. **Yarn Berry** (`.yarnrc.yml` with `npmRegistries[...].npmAuthIdent` raw username:password) and **Bun** (`bunfig.toml` `[install.registry] url/username/password`) need separate manual setup.

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
- `--repository` — Repository type: `oci` (default), `apk`, `java`, `python`, `javascript`. Each value implicitly binds the pull-token identity to the matching role (`registry.pull`, `apk.pull`, `libraries.java.pull`, etc.) so the token "just works" for the chosen ecosystem.
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
| `editor` | Modify most IAM/registry resources. **Surprise write exception**: `editor` has full create/delete/update on `subscriptions` (matches `owner`) — so editors can wire up webhooks without owner role. |
| `viewer` | Read-only across org resources |
| `console_viewer` | Console-only read; **no** `registry.pull` / `apk.pull`. Specifically lacks `apk.blobs` and `repo.blobs` get caps (so cannot pull blobs even though it can list manifests) and **cannot** create/update/delete event subscriptions. |
| `limited_owner` | Catalog Starter — viewer + pull-token creation. Has `identity.create`+`identity.list` AND `role_bindings.create`+`role_bindings.list` — so a limited_owner can self-issue pull tokens **and bind roles to identities**. No invites, no Custom Assembly. |
| `registry.pull` | Pull container images |
| `registry.push` | Push to repos |
| `registry.pull_token_creator` | Create pull tokens (has `identity.create` but deliberately no `identity.list` — pull-token creators cannot enumerate other identities in scope) |
| `apk.pull` | Pull from private APK repos (grants `apk.list`, `groups.list`) |
| `libraries.java.pull` / `libraries.python.pull` / `libraries.javascript.pull` | Pull from that ecosystem |
| `libraries.{java,python,javascript}.pull_token_creator` | Create pull tokens for that ecosystem (same `identity.list`-omission rationale as `registry.pull_token_creator`) |
| `guardener.admin` | Accept Guardener legal terms for the org (required once before anyone in the org can run sessions). Also has the capabilities of `guardener.user`. |
| `guardener.user` | Minimum role to run `chainctl agent dockerfile` sessions after terms are accepted. |

**UIDP structure:** `parent/child/suid`, where each segment is URL-safe hex. `UID` = 20 bytes, `SUID` = 8 bytes within a scope. Events, subscriptions, and `--identity`/`--parent` flags all expect UIDPs. Events propagate parent → child, so a subscription on a parent group sees events in its descendants; `Ce-Subject` on each event carries every SUID in the path.

**Discover the full capability matrix** with `chainctl iam roles capabilities list` (covers every resource and its valid actions). This is the canonical source when authoring custom roles or writing Terraform `chainguard_role.capabilities` blocks.

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
- `--type` / `--relationship` — Filter by identity type. Both spellings work (Options block lists `--type`; synopsis and examples use `--relationship`). Accepted values: `aws`, `claim_match`, `pull_token`, `service_principal`, `static`.
- `--expired` — Show only expired **static** identities (other types are not "expired" in the same sense).

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
- `--claim-pattern` — `stringArray` of custom claims to match. Repeatable; the published reference shows it as the only claim-matching flag. Example: `--claim-pattern=repository:your-org/your-repo --claim-pattern=event_name:push` (use regex for non-trivial patterns).
- `--issuer-keys` — JWKS-formatted public keys for the issuer (used for air-gapped/private Kubernetes clusters whose `/openid/v1/jwks` isn't reachable from Chainguard's STS — pass `--issuer-keys="$(kubectl get --raw /openid/v1/jwks)"`)
- `--expiration` — When issuer_keys expire (max 30 days, format: `yyyy-mm-dd`). For Kubernetes static identities this is the rotation deadline — re-run when the cluster rotates its JWKS.
- `--service-principal` — Service principal allowed to assume this identity
- `-f, --filename` — File with identity definition (YAML or JSON)
- `--role` — Role name or ID to bind this identity to (optional). Comma-separated for **multiple roles in one binding** (e.g., `--role=registry.push,registry.pull`).
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
- `--github-ref` — Single branch reference for the executing action (optional)
- `--github-refs` — **Regex** matching multiple refs (e.g. `--github-refs='refs/heads/main|master'`). Use instead of `--github-ref` when you need >1 branch.
- `--github-audience` — Audience for the GitHub OIDC token
- `--role` — Role name or ID to bind to (optional; comma-separated for multiple roles, e.g. `--role=registry.push,registry.pull`)
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

#### AWS native OIDC outbound identity federation (newer, preferred over `aws role` / `aws user`)
The `aws role` / `aws user` subcommands above use the legacy `GetCallerIdentity` + base64-encoded SigV4 request flow (the legacy token includes custom headers `Chainguard-Identity: <identity-id>` and `Chainguard-Audience: https://issuer.enforce.dev` — useful when debugging the legacy path). The current preferred path is generic `chainctl iam id create` against AWS's outbound OIDC issuer (`https://<uuid>.tokens.sts.global.api.aws`):
```bash
chainctl iam id create my-aws-id \
  --identity-issuer="https://<uuid>.tokens.sts.global.api.aws" \
  --subject="<aws-iam-arn>" \
  --role=registry.pull --parent <org>
```
The subject is the **literal IAM ARN** (e.g. `arn:aws:iam::123456789012:role/example`), not a pattern. Discover `iss` and `sub` from an actual token:
```bash
aws sts get-web-identity-token --audience=https://issuer.enforce.dev \
  --signing-algorithm=ES384 --query WebIdentityToken --output text \
  | jwt decode -j -
```
Then at login the AWS workload mints an OIDC token with `aws sts get-web-identity-token --audience=https://issuer.enforce.dev --signing-algorithm=ES384`. Required AWS IAM policy: `Action: sts:GetWebIdentityToken` with conditions `sts:IdentityTokenAudience=https://issuer.enforce.dev` and `sts:DurationSeconds<=300`.

#### Bitbucket Pipelines identity (Terraform-only — no native subcommand)
- **Issuer:** `https://api.bitbucket.org/2.0/workspaces/<workspace-name>/pipelines-config/identity/oidc`
- **Audience:** `ari:cloud:bitbucket::workspace/<workspace-uuid>`
- **Subject pattern:** `{<repository-uuid>}:.+` — the curly braces are literal (part of the token Bitbucket emits).
- Pipeline step must declare `oidc: true`; the OIDC token is available at `$BITBUCKET_STEP_OIDC_TOKEN`.

#### Buildkite identity (Terraform-only — no native subcommand)
- **Issuer:** `https://agent.buildkite.com`
- **Subject pattern:** e.g. `organization:<org>:pipeline:<pipeline>:ref:refs/heads/main:commit:[0-9a-f]+:step:.*`
- Job-time token: `buildkite-agent oidc request-token --audience issuer.enforce.dev`.
- A reusable `cgrauth` plugin (`.buildkite/plugins/cgrauth/hooks/pre-command`) is the canonical wrapper. Plugin lives **inside the same repo as the pipeline** (not a separate published plugin), with:
  - `plugin.yml` declaring `requirements: [bash]` and `configuration.properties.identity: { type: string }`
  - `hooks/pre-command` script that downloads chainctl, runs `buildkite-agent oidc request-token --audience issuer.enforce.dev`, then `chainctl auth login --identity-token "$token" --identity <UIDP>` and `chainctl auth configure-docker --identity-token "$token" --identity <UIDP>`.

#### Jenkins identity (chainctl + Terraform paths)
Created with the **generic** `chainctl iam identities create`, no `jenkins` subcommand:
```bash
chainctl iam identities create jenkins-ci \
  --identity-issuer https://YOUR_JENKINS/oidc \
  --subject jenkins \
  --description "Jenkins OIDC identity"
```
Requires the Jenkins "OpenID Connect Provider" plugin and a credential of kind "OpenID Connect id token". In a Jenkinsfile:
```groovy
withCredentials([string(credentialsId: 'jenkins-oidc', variable: 'IDTOKEN')]) {
  sh 'chainctl auth login --identity "$CHAINCTL_IDENTITY" --identity-token "$IDTOKEN"'
}
```

#### Keycloak identity (user CLI session, not CI)
Discover the values for an existing user:
```bash
curl --data "grant_type=password&client_id=<client>&username=<u>&password=<p>" \
  https://<keycloak>/realms/<realm>/protocol/openid-connect/token \
  | jq -r .access_token | cut -d. -f2 | base64 -d | jq .iss,.aud,.sub
```
Use `iss` as `--identity-issuer`, `sub` as `--subject`, `aud` (typically `account`) as `--audience`. Then login with `chainctl auth login --identity-token "$ID_TOKEN" --identity "$ID"`.

#### CircleCI identity (no ambient credentials)
```bash
chainctl iam identities create circleci-ci \
  --identity-issuer="https://oidc.circleci.com/org/<circleci-org-uuid>" \
  --subject-pattern="..." \
  --role=registry.pull --parent <org>
```

#### Microsoft Entra ID / Azure managed-identity (now the supported pattern)
The clean Azure pattern (per current docs) uses a **dedicated permission-free Entra app as the audience** plus a user-assigned **managed identity** attached to the workload. Earlier guidance to fall back to device-code (Entra federation issues access tokens, not ID tokens) is **superseded** by this flow:

```bash
# 1. Register a permission-free audience app
az ad app create --display-name chainguard-audience --sign-in-audience AzureADMyOrg \
  --identifier-uris "api://<client-id>" \
  --requested-access-token-version 2
az ad sp create --id <client-id>

# 2. Attach a user-assigned managed identity to your workload (VM / Container App / Function / etc.)

# 3. Create the Chainguard identity bound to it
chainctl iam id create azure-identity \
  --identity-issuer="https://login.microsoftonline.com/<tenant-id>/v2.0" \
  --subject=<principal-id-of-managed-identity> \
  --audience=<client-id-of-audience-app> \
  --role=registry.pull --parent <org>
```

At runtime the workload requests a token from IMDS:
```
GET http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=api://<client-id>&client_id=<managed-identity-client-id>
Metadata: true
```
Use the returned `access_token` as `--identity-token` for `chainctl auth login`.

For **Gov Cloud**, swap the issuer host to `https://login.microsoftonline.us/<tenant-id>/v2.0`.

#### Kubernetes pod identity (`--issuer-keys` and `--identity-issuer`)
Two paths depending on cluster reachability:
- **Internet-reachable cluster** — pass only `--identity-issuer` (Chainguard fetches JWKS from `<issuer>/.well-known/openid-configuration`). Discover issuer with `aws eks describe-cluster --query cluster.identity.oidc.issuer` (EKS), `az aks show --query oidcIssuerProfile.issuerUrl` (AKS), or the GKE format string. For ad-hoc inspection: `kubectl create token default | jwt decode -`.
- **Air-gapped / private cluster** — pass `--issuer-keys="$(kubectl get --raw /openid/v1/jwks)"` and `--subject` directly. This creates a **30-day static identity** that must be **rotated** whenever the cluster rotates its JWKS. Set `--expiration` to track the rotation deadline.

At runtime the pod uses a **projected service account token** with `audience: issuer.enforce.dev` (no scheme) and `expirationSeconds: 600` (10 min), mounted at `/var/run/chainguard/oidc/oidc-token`. It exchanges that token at `https://issuer.enforce.dev/sts/exchange?aud=https://console-api.enforce.dev&identity=<identity-id>` for a 1-hour Chainguard API token.

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
- `--ttl` — Duration the invite code will be valid. **Default `168h0m0s` (7 days).**

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

**Security caveat:** once a customer-managed IdP is configured, **any user who can authenticate with that IdP can access the Chainguard platform** — but having an IdP login does **not** automatically grant capabilities in any specific org; that's still a role-binding question. Confusion stems from "platform access ≠ org-level access". Keep either an OIDC default-provider bootstrap account OR an assumable identity as a fallback in case the custom IdP is misconfigured and locks you out.

**OIDC provider requirements:**
- Only **OIDC** is supported (no SAML). Only authorization-code grant — explicitly NOT client-credentials, device-code, or implicit.
- `openid`, `email`, `profile` scopes required (platform works partially with just `openid`); public unauthenticated OIDC discovery endpoint required.
- Redirect URI: `https://issuer.enforce.dev/oauth/callback`
- PKCE should be disabled or set to optional; restrict response types to authorization codes only.
- Actively supported integrations: Okta, Ping Identity, Keycloak, Microsoft Entra ID. Others via Generic Integration Guide (unsupported).

**Per-provider issuer URL cheat sheet:**
| IdP | Issuer URL pattern | Notes |
|-----|--------------------|-------|
| Okta | The Okta **org** URL (e.g. `https://<org>.okta.com`) | Configure under Sign On → "OpenID Connect ID Token" → set Issuer to the Okta org URL, **not** the auth-server URL. |
| Ping Identity | App's Configuration tab "Issuer URL" | Grant Type = Authorization Code with PKCE = Optional, Response Type = Code. |
| Keycloak | `https://<keycloak>/realms/<REALM_NAME>` | Client Type = "OIDC Connect" with Client Authentication ON. |
| Microsoft Entra ID (commercial) | `https://login.microsoftonline.com/${TENANT_ID}/v2.0` | App Registration must be **Single tenant** for org-only restriction. |
| Microsoft Entra ID (Gov Cloud) | `https://login.microsoftonline.us/${TENANT_ID}/v2.0` | |

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

### External Group Role Mappings
`chainctl iam external-group-role-mappings` (subcommands: `create`, `delete`, `list`)

Maps **external IdP groups → Chainguard roles** so role assignment becomes dynamic: a user gains/loses a role automatically as their group membership changes upstream, instead of binding each identity one at a time. Requires a custom OIDC identity provider (`chainctl iam identity-providers`) whose tokens emit a **group claim**; on login Chainguard reads the claim and grants the mapped role to any user in a matching group.

| Command | Description |
|---------|-------------|
| `create` | Map an external group to a role at a scope. Flags: `--external-group-id` (the IdP group identifier / claim value), `--idp` (identity provider UIDP that owns the mapping), `--role` (role UIDP or name to grant), `--scope` (group UIDP — the org root — where the role applies). |
| `delete` | Delete a mapping. Positional `MAPPING_ID`; flag `-y, --yes`. |
| `list` | List mappings. Flags: `--parent` (required; org/folder UIDP), `--idp` (narrow to one IdP UIDP). |

```bash
# Grant the "viewer" role to everyone in the IdP group "platform-eng"
chainctl iam external-group-role-mappings create \
  --external-group-id=platform-eng --idp=<idp-uidp> \
  --role=viewer --scope=<org-uidp>

chainctl iam external-group-role-mappings list --parent=my-org
```

### Account Associations
`chainctl iam account-associations` (aliases: `accountassociations`, `account-association`, `accountassociation`; subcommand aliases: `describe`/`desc`/`get`)

Configure cloud provider account associations (AWS, Azure, GCP). Cloud-side federated identity trust must be pre-configured before `set` succeeds — `check` probes the cloud-side readiness.

| Command | Description |
|---------|-------------|
| `describe` | Describe cloud provider account associations for a location. **Target is a positional argument** (`ORGANIZATION_NAME|ORGANIZATION_ID|FOLDER_NAME|FOLDER_ID`) — not `--parent`. Caps: `groups.list`, `account_associations.list`. Flags: `--aws`, `--chainguard` (service principal), `--gcp`. |
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

Chainguard images come in two broad **entitlement** categories:
- **Free/public** (`cgr.dev/chainguard/<name>`) — only `:latest` and `:latest-dev`. Mirrored to Docker Hub. No entitlement required.
- **Production/entitled** (`cgr.dev/<your-org>/<name>`) — full multi-version tags, FIPS, Unique Tags, and Custom Assembly enabled.

The catalog itself is now organized into **five named categories** (visible in the Console / Directory and accepted by `--tier`): **Free, Base, Application, FIPS, AI**. "Production" is the umbrella for everything non-Free; Base/Application/FIPS/AI are sub-buckets of it.

**Multi-layer image architecture (May 2025).** Chainguard switched from single-layer apko images to multi-layer "per-origin" images for better cache efficiency. Doc claim: "A ~70% reduction in the total size of unique layer data across our image catalog compared to the single-layer approach, A 70-85% reduction in the cumulative bytes transferred…". Practical impact: a pull-heavy node fetching multiple Chainguard images shares layers across them; CI pulls benefit most.

**ISA baselines.** Chainguard images are built against specific CPU baselines, not lowest-common-denominator:
- `x86_64`: **x86-64-v2** (Sapphire Rapids and equivalents) — pre-Nehalem hosts can't run these.
- `AArch64`: **Armv8-A with CRC and Cryptographic extensions** (Neoverse V2 baseline).

If a Chainguard image hard-faults at startup on an older host, ISA mismatch is the first thing to check.

**OCI annotation semantics on Chainguard images:**
- `org.opencontainers.image.created` is "calculated from the build time of the most recently built package within the container image" — it is **not** when the image manifest was assembled. Don't use it as a build-time proxy in CI gating.
- `dev.chainguard.package.main` may change between versions of the same image, and may be empty or unset. Consumers parsing it should defend against an empty string.

**Tag types** (exposed via `images list` flags and mentioned here for reference):
- `latest` / `latest-dev` — rolling; `-dev` includes shell, apk, dev utilities.
- **Variants** — some images also ship `-slim` (even smaller than distroless) or `-fips` (FIPS-validated).
- **Epoch tags** — `1.2.3-r4`, where `-rN` is the Wolfi/apk package epoch.
- **Date tags** — `latest-{date}` and `<version>-{date}` (shown via `--show-dates`).
- **Referrer tags** — `sha256-<digest>.sig|sbom|att` (Cosign-style, shown via `--show-referrers`).
- **Unique Tags** (opt-in, private only) — timestamped like `<version>-YYYYMMDDHHMM` (12-digit minute precision; may also carry a name prefix, e.g. `openjdk-17-202412120223`). Docs also show full second-precision examples like `1.2.3-20260218175623` (14-digit), so consumers should treat the suffix length as variable. Operational gotchas: enables org-wide unless you exclude per-repo; **disables semantic-tag updates** (`1.2.3` no longer receives rebuilds — you get `1.2.3-YYYYMMDDHHMM` instead); tag-list responses can grow large enough to cause registry-client performance issues; tags are not registry-level immutable (a mirror can still push-over them). Chainguard recommends **digest pinning** (`@sha256:...`) over Unique Tags for true immutability.

**Spellings:** both `chainctl images` (plural) and `chainctl image` (singular) work; same for `repos` / `repo`. The published reference uses plural; some how-to docs use singular. Either copy-paste lands you in the right command.

**Chainguard VMs** are a separate product (delivered via AWS/GCP/Azure marketplaces, qcow2, VMDK) — **not managed via `chainctl`**. There is no `chainctl vms` namespace.

### `chainctl images list`
List tagged images from Chainguard registries. Aliases: `ls`.

**Flags:**
- `--parent` — Name or ID of parent location
- `--public` — List repos from public Chainguard registry
- `--recursive` — Search recursively through all descendants
- `--repo` — Search for a specific repo by name
- `--active-only` — Show only active (not deprecated/EOL) tags
- `--show-dates` — Show date tags (e.g. `latest-{date}`)
- `--show-epochs` — Show epoch tags (e.g. `1.2.3-r4`)
- `--show-referrers` — Show referrer tags (e.g. `sha256-deadbeef.{sig,sbom,att}`)
- `--updated-within` — Filter by update recency (0 disables)

**Required Capabilities:** `groups.list`, `repo.list`, `tag.list`

**Interactive behavior:** if `--parent` is omitted and the account belongs to multiple orgs, chainctl shows an arrow-key org picker before listing. The tree output uses `├ [name]`, `│ ├ sha256:...`, `│ │ ├ [tag]`. To skip the picker in scripts, always pass `--parent`.

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
- `-o, --output` — **Defaults to `json`** (not global default). Accepts `json`, `go-template`, `markdown`.
- `--template` — Go template for `--output=go-template`
- `--template-file` — Path to Go template file

**Output shape:** top-level keys `packages` (with `added`/`removed`) and `vulnerabilities` (present in first image, absent in second). Order matters — the first image is FROM, the second is TO; swapping them inverts `added`/`removed`. PURLs are grouped without their `@version` component; duplicate PURLs fold to the first one found. When non-`apk` artifact types are included, entries look like `pkg:oci/index@sha256:...?mediaType=...` and `pkg:oci/image@sha256:...?arch=amd64&os=linux&mediaType=...`. **Output format is officially "subject to change"** per the docs — pin parser logic to specific keys, not whole-output shape.

**Required Capabilities:** `identity.list`

### `chainctl images history <image>`
Show history for a specific image tag.

**Flags:**
- `--parent` — Organization to view from
- `--recursive` — Search recursively through descendants

**Required Capabilities:** `groups.list`, `repo.list`, `tag.list`, `manifest.metadata.list`

**Interactive behavior:** if you omit the tag (`chainctl images history nginx`), chainctl shows an arrow-key picker over available tags. With a tag, it returns the reverse-chronological digest history; for multi-arch images, per-architecture digests + sizes are returned.

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

- `list` — List tags from repositories (caps: `groups.list`, `repo.list`, `tag.list`). Flags: `--parent`, `--public`, `--repo`, `--active-only` (omit deprecated/EOL tags), `--all` (return all tags matching the digest of the specified image ref), plus the same show-dates/show-epochs/show-referrers/updated-within flags as `images list`.
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
- `--tier` — Catalog tier: `BASE`, `APPLICATION`, `FIPS`, `AI`, `COMMERCIAL`, `DEVTOOLS`. Available on `create` as well as `update`. (The public categories doc only enumerates Base/Application/FIPS/AI; the CLI and API accept the broader superset above, plus `UNKNOWN` in API responses.)
- `--bundles` — Comma-separated list of bundles to assign

**Organization catalog cap:** Chainguard enforces a **maximum of 2500 container image repositories per organization** (Catalog Pricing limit). Creating a 2501st repo fails server-side.

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
- `--tier` — Catalog tier (same values as `create`: `BASE`, `APPLICATION`, `FIPS`, `AI`, `COMMERCIAL`, `DEVTOOLS`)
- `--expiration` — Sync expiration time (e.g. `1969-12-31`)

**Renaming caveat:** changing `--name` breaks every client that pulls the old name. Don't rename a production repo without a migration plan.

**Required Capabilities:** `groups.list`, `repo.update`, `repo.list`

### Custom Assembly (chainctl images repos build)

Custom Assembly lets you customize any entitled Chainguard image by adding packages, environment variables, OCI annotations, custom user accounts/groups, and custom certificates — without forking images or maintaining custom build pipelines. The YAML schema is an **apko overlay file** (strict subset of apko config). The customized image is built on the **Factory** (Chainguard's GitHub-Actions-on-GKE build system) and shipped with full build logs, SBOM, and provenance metadata. Builds for Custom Assembly outputs are signed by per-org `CATALOG_SYNCER` and `APKO_BUILDER` identities (not the `chainguard-dev/mono` workflow that signs public images), so cosign verify needs `--certificate-identity` pointing at those.

**Required Capabilities:** `groups.list`, `repo.create`, `repo.update`, `manifest.create`, `tag.list`, `apk.list`, `build_report.list`

#### YAML Configuration Sections

Custom Assembly uses an apko-overlay YAML with these sections:

| Section | Description |
|---------|-------------|
| `contents.packages` | Additional packages to install (must be in Chainguard's package repo). Managed cert bundles (`ca-certificates-aws-rds-global`, `ca-certificates-aws-rds-govcloud-global`, `ca-certificates-dod-eca`, `ca-certificates-dod-wcf`) go here, not under `certificates`. |
| `contents.runtime_repositories` | **Custom runtime APK repositories** — a list of internal mirror URLs that **replace** the default `virtualapk.cgr.dev` entries in `/etc/apk/repositories` in the assembled image, so runtime `apk add` inside the container resolves against your infra instead of Chainguard's endpoints. E.g. `runtime_repositories: ["https://apk-mirror.example.com/chainguard"]`. |
| `environment` | Environment variables for the image (`CHAINGUARD_` prefix is reserved). Values must be strings — quote numeric values (`PORT: "3000"`). |
| `annotations` | Custom OCI annotations. **Reserved prefixes — cannot be used at all**: `dev.chainguard` and `org.opencontainers`. |
| `accounts` | Custom users/groups. Users accept `username`/`uid`/`gid`/`homedir`. Groups accept `groupname`/`gid`/`members` (list of usernames). `run-as` accepts a UID or username string. |
| `certificates` | Custom inline PEM certs (`certificates.additional[].name`, `.content`). Each entry holds exactly one PEM block; freeform descriptive text above the `-----BEGIN CERTIFICATE-----` marker inside `content:` is allowed and ignored. Per-cert file is written to `/usr/local/share/ca-certificates/<name>.crt`. |

**SLA, FIPS, and tag semantics for customized images:**
- The CVE-remediation SLA applies to the base image; the customized image benefits from it because the added packages are entitled and scanned daily. Wolfi-OSS packages added (beta full-Wolfi private APK) are **not covered**.
- The Chainguard FIPS Commitment **does not apply** to Custom Assembly outputs, even when based on a `-fips` image. Customizing a FIPS image does not guarantee FIPS-mode by Chainguard.
- Semantic tags (`v1.2.3`) on a customized image always reflect the **base** image version. Added-package updates trigger auto-rebuilds that re-point `latest`; the semantic tag stays pinned to the same base digest until the base itself updates.
- Common build failure pattern: adding newer packages to an older base — the newer package may have been built against a newer glibc than the base provides.

#### `chainctl images repos build edit`
Interactive editor to customize a Chainguard image. Opens your editor with the current config (or a template for new repos). Shows a diff for review before applying.

**Flags:**
- `--repo` — Name or ID of the repo to edit
- `--parent` — Name or ID of the parent location
- `-f, --file` — Pre-written config file (skips interactive editor)
- `--save-as` — Create a new repo with the edited config instead of modifying the existing one
- `--with-certificates` — Certificate file to include. **Repeatable** (not comma-separated) — pass once per file. Currently the only chainctl + API path for managing custom certs (Console UI is not yet GA).

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
Apply a YAML configuration file non-interactively to an **existing** repo. Ideal for CI/CD pipelines and GitOps workflows. **Cannot create new repos — `--save-as` is `edit`-only per the Custom Assembly docs.** To create a new repo from a file non-interactively, use `chainctl images repos build edit --file --save-as` (no `--yes` flag on `edit`, but `--file` already bypasses the interactive editor).

**Flags:**
- `--repo` — `stringArray`, **repeatable**, accepts shell-style wildcards (`*`, `?`, `[abc]`). Fan one config out over many repos in one invocation. Can be specified multiple times.
- `--parent` — Name or ID of the parent location
- `-f, --file` — Config file to apply
- `--with-certificates` — Certificate file to include. The `edit` reference describes it as "can be specified multiple times" (repeatable); the `apply` reference Options block calls it a "Comma separated list" — both forms are accepted by the underlying cobra `stringSlice`. At least one of `--file` or `--with-certificates` is required.
- `--dry-run` — Print the diff without applying. **Exits non-zero if changes would be made** — useful for CI drift detection.
- `-y, --yes` — Auto-confirm (for CI/CD)

**Batch mode:** when `--repo` matches multiple repos (via wildcard or repeated flag), builds run up to 10 in parallel; one confirmation prompt covers the whole batch; final output is a per-repo results summary.

**Required Capabilities:** `groups.list`, `repo.update`, `repo.list`

**Examples:**
```bash
# Apply config from a file (update existing repo)
chainctl images repos build apply --repo=my-custom-python --file=config.yaml

# Create a new repo from a file (uses edit, not apply — --save-as is edit-only)
chainctl images repos build edit --repo=python --file=config.yaml --save-as=my-custom-python --parent=my-org

# CI/CD: apply with auto-confirm
chainctl images repos build apply --repo=my-custom-python --file=config.yaml --yes

# Apply with custom certificates
chainctl images repos build apply --repo=my-custom-python --file=config.yaml --with-certificates=ca1.pem --with-certificates=ca2.pem

# CI drift check (exits non-zero if anything would change)
chainctl images repos build apply --parent=my-org --repo=my-custom-python --file=config.yaml --dry-run

# Batch: apply one config across every repo in an org
chainctl images repos build apply --parent=my-org --repo='*' --file=base-policy.yaml --yes
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

### `chainctl images helm refs <CHART_REFERENCE>`
List every distinct image reference pinned in a Chainguard Helm chart's **chart-lock attestation**, including images pulled in by subcharts. Distinct from `helm values` (which generates relocation overrides). Positional `CHART_REFERENCE` is required (e.g. `cgr.dev/my-org/charts/flux:v2.18.4`).

**Flags:**
- `--repository` — Override the `{registry}/{org}` prefix on each ref (for relocated/mirrored copies). Does **not** affect `-o json` output.

**Output:** default is one ref per line as `{registry}/{repoName}:{tag}@{digest}`; `-o json` emits each ref as an object with `repoName`, `tag`, `digest`.

```bash
chainctl images helm refs cgr.dev/my-org/charts/flux:v2.18.4
chainctl images helm refs cgr.dev/my-org/charts/flux:v2.18.4 --repository myregistry.internal/images/chainguard -o json
```

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

**Full CloudEvent envelope** carries: `Ce-Type` (event type), `Ce-Source` (`cgr.dev` for registry events, otherwise the `console-api.enforce.dev` endpoint URL), `Ce-Audience: customer`, `Ce-Group: <UID of parent group>`, `Ce-Subject` (UIDP including every SUID in the path — for subscriptions inside sub-folders the sub-group SUID is appended), `Ce-Actor` (e.g. `enforce-prod-registry-<id>@prod-enforce-fabc.iam.gserviceaccount.com` — the GCP service account that emitted the event), `Ce-Specversion`, `Ce-Time`, `Authorization: Bearer <JWT>`, `User-Agent: Chainguard Enforce`. **Body of `registry.pull` events** carries `location` (approximate geolocation like `"ColumbusOHUS"` or `"Minato City13JP"`), `remote_address`, and `user_agent` — useful for audit/anomaly detection. Verification order: validate `iss`, validate `sub`, then verify JWT signature.

**Container Mirroring example:** Chainguard publishes a Terraform module at `platform-examples/image-copy-gcp/iac` that wires `registry.push` events into a webhook that mirrors images to GCP Artifact Registry. Inputs: `name`, `project_id`, `group_name`, `location`, `dst_repo`. Result path: `<location>-docker.pkg.dev/<project_id>/<name>-<dst_repo>`.

**Common event types (`Ce-Type`):**
- `dev.chainguard.registry.pull.v1` / `push.v1`
- `dev.chainguard.api.auth.registered.v1`
- `dev.chainguard.api.events.subscription.created.v1` / `.deleted.v1`
- `dev.chainguard.api.iam.{group,group_invite,identity,identity_providers,rolebindings,account_associations}.{created,updated,deleted}.v1`
- `dev.chainguard.api.iam.rolebindings.created.batch.v1` — batch variant emitted when many role-bindings are created in one call. Webhook handlers regexing on `Ce-Type` need to include this.
- `dev.chainguard.api.iam.terms.accepted.v1` — emitted when an org accepts legal terms (Guardener, Skills, etc.).
- `dev.chainguard.api.platform.registry.{repo,tag}.{created,updated,deleted}.v1`
- `dev.chainguard.api.platform.registry.chart.added.v1` — a Helm chart was added to the registry.
- `dev.chainguard.api.platform.registry.policies.{created,updated,deleted}.v1` and `...bindings.{created,updated,deleted}.v1` — image-policy and policy-binding lifecycle (the `chainctl policies` feature).

**Chainguard Notifications vs CloudEvents:** "Chainguard Notifications" (Console → Slack/email, sent by Customer Success for breaking changes / EOL / incidents) is a **separate** feature from CloudEvents subscriptions. Users often conflate them. Notifications have **three categories**: Breaking changes, Incidents, and Product lifecycle (End of life / New releases). In-app delivery to the Activity Center is automatic with no config; email is opt-in per customer; Slack is opt-in per channel. **Only the `owner` role can configure notifications.** Private Slack channels require **adding the Chainguard Notifications app inside Slack first** (Channel → Integrations → Add an App → Chainguard Notifications) before they show up in the Console dropdown.

---

## packages — Package Management

`chainctl packages` (aliases: `package`, `pkg`, `pkgs`)

### `chainctl packages versions list <PACKAGE_NAME>`
List version data for a Chainguard **managed version stream**. Aliases: `ls`.

**Usage:** `chainctl packages versions list <PACKAGE_NAME>` — `PACKAGE_NAME` is a **required positional argument** (e.g. `bazel`, `airflow`, `python-3.13`, `actions-runner`). These are curated version streams, not arbitrary apk packages. For arbitrary apk packages, run `apk info` / `apk search` inside an image instead.

**Flags:**
- `--include-inactive` — Include packages within the EOL grace period end date
- `--show-active` — Show only active versions
- `--show-eol` — Show only EOL versions
- `--show-fips` — Show only FIPS versions

**Required Capabilities:** `version.list`

---

## actions — Chainguard Actions product entitlements

`chainctl actions entitlements` — manage the Chainguard Actions product entitlement on an organization.

| Command | Description |
|---------|-------------|
| `create --parent=<org>` | Enable Actions for an organization. |
| `delete --parent=<org>` | Disable Actions for an organization. |
| `list --parent=<org>` | List Actions entitlements for an organization. |

`--parent` is the only command-specific flag (each takes `[--output=json|table]`). All three are simple toggles — no policy options.

### `chainctl actions catalog list`
Browse the **public** Chainguard Actions catalog (no org context).

**Flags:**
- `--upstream-owner` — Filter to actions mirroring this upstream owner (e.g. `actions`).
- `--upstream-repo` — Filter to a specific upstream repo (**requires `--upstream-owner`**).
- `--page-size` — Max actions per page (`0` = server default).

### `chainctl actions list`
List Actions catalog entries available **within an organization**.

**Flags:**
- `--parent` — Org name or ID to list actions for (required).
- `--upstream-owner` / `--upstream-repo` — Same upstream filters as `catalog list` (`--upstream-repo` requires `--upstream-owner`).
- `--page-size` — Max actions per page.

### `chainctl actions discover [TARGET]`
Walks a GitHub repo's workflows and composite-action definitions and resolves every action and container image they (transitively) use. Useful for auditing supply-chain dependencies before enabling Chainguard Actions.

**Positional `TARGET`:**
- Local directory (default `.`).
- `owner/repo` — a GitHub repo at default branch.
- `owner/repo[/subpath]@version` — a specific ref (tag, branch, commit).

**Flags:**
- `--cache-dir` — Default `$TMPDIR/chainctl-discover-cache`.
- `--clear-cache` — Wipe the cache before discovering.
- `--timeout` — Default `5m`.

**Auth:** requires either `$GITHUB_TOKEN` in the environment or a logged-in `gh` CLI session (the command will call `gh auth token`).

---

## policies — Registry policy gates

`chainctl policies` is a registry governance feature that controls which images your org can pull (guardrails evaluated against each image at pull time). **The namespace is `policies` (plural) — there is no `chainctl policy-gate` command.** **Availability is opt-in via Customer Success** — not on by default. Each *binding* attaches a *policy* to a set of resources with a *mode* (`ENFORCE` blocks non-compliant pulls; `DRY_RUN` only logs). Default mode is `DRY_RUN`. (Note: the chainctl reference Options block for `enable` mentions a `LOG` value, but synopsis, examples, and the parent concept doc all use `ENFORCE` / `DRY_RUN` — treat the Options-block wording as stale.)

| Command | Description |
|---------|-------------|
| `list --parent=<org>` | List policies. |
| `describe --policy=<name> --parent=<org>` | Show a policy's full definition + parameter schema (output is usable with `enable`). |
| `enable --policy=<name> --parent=<org>` | Enable a policy (creates a binding with the given mode). Shortcut for `binding create`. |
| `disable --policy=<name> --parent=<org>` | Disable a policy. Shortcut for `binding delete`. |
| `check IMAGE_REF` | Check whether an image (tag or digest) would be allowed by the current policies. **Positional** image ref; non-zero exit on `DENIED`/`ERROR` (CI-friendly). |
| `binding create` | Create a new binding (alternative to `enable`). |
| `binding delete [BINDING_ID \| --policy <name> --parent <org>]` | Delete a binding by ID (positional) or by policy. |
| `binding list --parent=<org>` | List bindings. |

**Common flags on `enable` / `binding create`:**
- `--policy` — Policy name or UIDP to attach (required).
- `--parent` — Org or folder for the binding.
- `--mode` — `ENFORCE` or `DRY_RUN` (default `DRY_RUN`).
- `--param` — `stringArray`, repeatable; parameter values as `key=value` (the schema comes from `policies describe`).
- `--resources` — `strings`; default `[registry.chainguard.dev/Repo]`. The resource types the binding applies to.

---

---

## skills — Skills Registry (`skills.cgr.dev`)

`chainctl skills` publishes and consumes agent skills as OCI artifacts at `skills.cgr.dev/<org>/<name>:<tag>`. Useful when an org wants to ship internal Claude / agent skills through Chainguard's registry plumbing.

| Command | Description |
|---------|-------------|
| `accept-terms` | Accept the legal terms required to publish skills. **An org owner must run this once per org before `push` works.** Flags: `--group` (org name/UIDP; interactive picker if omitted), `--yes` (non-interactive — confirms agreement to the docs at `https://www.chainguard.dev/legal/agent-skills-disclosure`; needed in CI without a TTY). Re-running after acceptance is a no-op. |
| `push [<path>]` | Package a directory containing `SKILL.md` and publish. Flags: `--dry-run`, `-g, --group` (org; defaults to current context), `-t, --tag` (default `"latest"`). |
| `pull <ref> [<dir>]` | Download a published skill. `<ref>` = `org/name[:tag]`; `<dir>` defaults to `./`. Flag: `--force`. |
| `install <ref>` | Download AND install into agent directories — writes a shared canonical copy to `.agents/skills/` and creates agent-specific symlinks. Flags: `-a, --agent stringArray` (repeatable target list; `--agent '*'` for all), `--copy` (per-agent copies instead of symlinks), `--global` (install to `~/` instead of project-local). |
| `uninstall <name>` | Local-only (does NOT touch the registry). `<name>` is the skill name (no org/tag). Flags: `-a, --agent stringArray`, `--global`, `-y, --yes`. |
| `list` | List skills published by an org. Flag: `-g, --group`. |
| `versions <ref>` | List all published tags for `<ref>` = `org/name` (no tag). |
| `describe <ref>` | Show metadata for a published skill. |
| `validate [<path>]` | Spec-compliance check, no network. Flag: `--strict` (also warns on optional recommended fields). |
| `delete <ref> [-y]` | Delete a tag. Must include the tag (`org/name:tag`). Deleting `latest` requires extra confirmation. |
| `entitlements create --parent=<org>` | Enable the org to push/pull via `skills.cgr.dev`. **Idempotent** — re-running on an already-entitled org converges without error. |
| `entitlements delete --parent=<org>` | **DESTRUCTIVE** — removes the skills folder and entitlement; **all skill repos under the org become inaccessible with no recovery path.** Always confirm before running. |
| `entitlements list --parent=<org>` | List Skills Registry entitlements. |

---

## starter — Catalog Starter orgs

`chainctl starter` manages **Catalog Starter** orgs — a free-tier surface that grants evaluation access to a small set of Chainguard images, scoped to a verified `kind=starter` org tied to the user's email domain. Takes no command-specific flags (only global inherited ones).

**Self-Serve Catalog Experience (separate product).** Beyond Starter, paid Catalog Pricing customers can self-serve "Add image" from the Console (or via `chainctl images repos create`) to bring in any image from Chainguard's catalog. Requires a role with `repo.create`, `repo.list`, `repo.update`, and ideally `registry.entitlements.list` — only built-in `owner` has all by default. Rename a self-served repo with `chainctl images repos update <name> --parent=<org> --name=<new>` — **warning: renaming breaks every client that pulls the old name**.

| Command | Description |
|---------|-------------|
| `init` | Interactive — authenticates with a supported IdP, registers a Chainguard identity for the user's email, and creates a new starter org tied to the email domain. **Business email only** (Gmail/Yahoo refused). Auth must be email+password OR Google (the latter only if the business uses Google Workspace on its own domain). If the org already exists, fall back to `chainctl starter request-access`. |
| `request-access` | Sends an access request to the existing starter org for the authenticated email's domain. **Returns identical response whether or not a matching org exists** (privacy by design — don't infer existence from the response). |
| `add-images IMAGE_NAME [IMAGE_NAME ...]` | Adds catalog images to the caller's auto-discovered starter org. Total cap is server-enforced. |
| `status` | Shows registry path, account provisioning status, image quota usage, and per-image readiness: `INITIALIZING` until the catalog syncer has created the repo AND mirrored ≥1 tag, `READY` afterwards. |

---

## libraries — Ecosystem Libraries

`chainctl libraries` (aliases: `libs`, `ecosystems`)

**Two endpoints back the libraries surface** (the skill's `libraries.cgr.dev` callouts refer to the first):
- `https://libraries.cgr.dev/<ecosystem>/` — Chainguard-built artifacts with policy controls.
- `https://libraries.cgr.dev/<ecosystem>-upstream/` — pure upstream-mirror endpoint (e.g. `/javascript-upstream/`). Used directly for Yarn Classic auth blocks, GAR second-remote-repo setups, and as the tarball host in lockfiles after `update-hashes`. Blob storage 302-redirects to `9236a389bd48b984df91adc1bc924620.r2.cloudflarestorage.com` (same R2 host the container registry uses) — any HTTP-caching reverse proxy in front of `libraries.cgr.dev` can poison itself by caching the redirect target.

### `chainctl libraries verify`
Analyze artifacts to determine how much was built from source by Chainguard, using signature-based binary identification with a checksum fallback. Aliases: `check`.

**Supports:** directories, archives, packages, container images (registry refs, local images, `docker-archive:` format). Binary formats: **JAR, WAR, EAR, ZIP, TAR, WHL, APK, npm `.tgz`, container images**. Package-manager caches auto-detected: npm (`_cacache/index-v5/`), pnpm v10+ (`v10/index/`, `v11/index/` — **pnpm v9 and earlier are not supported** because v9 doesn't record the tarball hash in the index path), Yarn Classic (`yarn:` prefix; alt cache path: `chainctl libraries verify yarn:~/Library/Caches/Yarn/v6`).

**Path prefixes:** `remote:<url>` analyzes a remote artifact without downloading (anonymous/public URLs only — no authenticated repo access); `localhost/<image>` analyzes a local container image (distinct from `docker-archive:`); `./node_modules` works **only if `.package-lock.json` is present** in the directory (npm v7+; images built before that or with the lockfile stripped can't be verified through this path).

**Flags:**
- `-d, --detailed` — Show detailed per-artifact results
- `--no-color` — Disable colored output
- `-o, --output` — Output format: `text` (default), `json`, `yaml`
- `--verbose` — Enable verbose output
- `--parent` — Parent organization for authentication. If your account is in multiple orgs you'll need this on every `libraries` command, or set a default with `chainctl config set default.group <org>`.
- `--ecosystems-url` — URL for the Ecosystems Proxy (default `https://libraries.cgr.dev`)

**Output:** reports a **Verification Coverage percentage** per artifact (e.g. 100.00% = fully from Chainguard sources, 0% = none). Uses cosign-verified SLSA attestations signed against `issuer.enforce.dev`, with signature-based identification falling back to checksums.

**Bundled-output limitation:** fat/uber/shaded JARs always return 0%. **The same limitation applies to other ecosystems where dependencies are bundled into a single output artifact, such as JavaScript bundles and Python applications packaged with tools that inline dependencies.** Verify individual artifacts from `~/.m2/repository` / `node_modules` / `.venv` *before* assembly.

**`chainver`** is a separate Java-focused verification tool referenced in the Java management docs. The skill recommends `chainctl libraries verify` as the unified tool — mention `chainver` only if a user asks about it specifically.

**Manual verification path (cosign verify-blob-attestation, JavaScript)** when you need to inspect a single tarball outside `chainctl`:
```bash
# fetch the attestation bundle
curl -H "Authorization: Bearer $(chainctl auth token --audience=libraries.cgr.dev)" \
  "https://libraries.cgr.dev/javascript/-/npm/v1/attestations/PACKAGE@VERSION" \
  | jq '.attestations[] | select(.predicateType=="https://slsa.dev/provenance/v1")' > bundle.json

cosign verify-blob-attestation --bundle bundle.json --new-bundle-format \
  --certificate-oidc-issuer=https://issuer.enforce.dev \
  --certificate-identity-regexp="^https://issuer.enforce.dev/" \
  --check-claims=false PACKAGE-VERSION.tgz
```

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

### `chainctl libraries update-hashes [<lockfile>]`
Rewrite integrity hashes in an existing lockfile to use Chainguard checksums. **Preferred over delete-and-relock when a lockfile already exists.** Auto-detects the lockfile in the current directory if no path is supplied.

**Supported formats:**
- JavaScript: `package-lock.json` (npm v2/v3), `yarn.lock` (Classic and Berry), `pnpm-lock.yaml`, `bun.lock`.
- Python: `requirements.txt` (pip-tools `--hash`), `poetry.lock`, `pdm.lock`, `uv.lock`, `pylock.toml` (PEP 751), `Pipfile.lock`.

**Flags:**
- `--cuda` — CUDA variant to include alongside Python wheels (e.g. `cu124`, `cu128`, `cu130`).
- `--dry-run` — Show what would change without writing.
- `--ecosystem` — `auto`, `js`, or `python` (default `auto`; detected from lockfile name).
- `--ecosystems-url` — Default `https://libraries.cgr.dev`. The command automatically appends `/javascript/{name}/{version}` for JS and `/{python,python-remediated,cu###}/simple` for Python.
- `--no-color`
- `--parent` — Parent organization for authentication.
- `--remediated` — Use the `python-remediated` registry (CVE-patched packages, Python only).
- `--replace` — Replace integrity hashes instead of appending. **No-op for `uv.lock` / `pdm.lock` / `pylock.toml`** — those formats store one hash per artifact and always replace.
- `--registry-url` — Point at a private repository manager (Artifactory / Nexus / GAR) instead of `libraries.cgr.dev` directly. **Mutually exclusive with `--ecosystems-url`, `--remediated`, and `--cuda`.** When set, Chainguard token sources are **deliberately not consulted** to avoid leaking a JWT to a third-party host.
- `--fallback-registry-url` — For JS only. Synthesizes tarball URLs for packages not found in Chainguard Libraries (e.g. when only a subset of the dependency tree is Chainguard-rebuilt). Empty by default; the command fails listing offenders if fallback is needed. Avoid pointing at `registry.npmjs.org` directly (malware risk) — prefer a private/internal mirror.
- `--token`, `--username`, `--password` — Auth credentials used with `--registry-url`. Also readable from env vars `CHAINCTL_AUTH_TOKEN`, `CHAINCTL_REGISTRY_USERNAME`, `CHAINCTL_REGISTRY_PASSWORD`, and from `~/.netrc` (or `$NETRC`) keyed on the registry host.

**Behavior:** for formats that accept multiple hashes (pip-tools `requirements.txt`, `poetry.lock`), Chainguard hashes are **appended** alongside upstream hashes; for single-hash formats, they replace. Tarball/`resolved` URLs are also updated. Output prints a "Next steps" block with the right reinstall command for the detected toolchain — surface it to users so they don't miss the post-rewrite reinstall. Caveats: JS packages resolved through the upstream-fallback path may keep `registry.npmjs.org` URLs until Chainguard rebuilds them; `--registry-url` enables repository-manager workflows that previous releases didn't support.

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
- `--policy` — Policy to apply: `CHAINGUARD` (Chainguard-only) or `CHAINGUARD_AND_UPSTREAM` (Chainguard repo with upstream fallback; **documented only for JAVASCRIPT**). **Default `chainguard`.** Case-insensitive — examples in the reference use lowercase (`chainguard`, `chainguard_and_upstream`). Listing shows the canonical names as `POLICY_CHAINGUARD` / `POLICY_CHAINGUARD_AND_UPSTREAM`.

**Cooldown is no longer set here.** Older chainctl exposed `--cooldown-days` on this command; as of `v0.2.291` cooldown duration moved to the separate `chainctl libraries policy` subsystem (see below). `entitlements create` now only controls the ecosystem and the Chainguard-vs-upstream policy.

**`create` is an upsert.** There is no separate `update` subcommand — to change the Chainguard-vs-upstream policy on an existing entitlement, rerun `create` with the new value for the same `--parent` + `--ecosystems`. (To change the cooldown, use `chainctl libraries policy`.)

**Required Capabilities:** `groups.list`, `libraries.entitlements.create`

**Ecosystem-specific caveats:**
- **No SLA on `libraries.cgr.dev`** — proxy through an artifact manager (Artifactory, Nexus) for production.
- **Chainguard checksums differ from upstream.** Lockfiles pinned to upstream hashes will fail strict-verification — use `chainctl libraries update-hashes` (preferred) or relock after adoption.
- **Cooldown applies symmetrically.** When `CHAINGUARD_AND_UPSTREAM` is enabled, the cooldown holds back brand-new **Chainguard-built** versions too (not just upstream-fallback ones), so dependency trees resolve consistently across both sources.
- **Java**: no snapshot versions, no source/Javadoc JARs, no distribution tarballs in most cases. Must purge `~/.m2/repository` AND `~/.gradle/caches/` and any `mavenLocal()` before first resolve. Env var idiom: `CHAINGUARD_JAVA_IDENTITY_ID` / `CHAINGUARD_JAVA_TOKEN`. The recommended `~/.m2/settings.xml` defines Chainguard **and** re-declares `central` with `<snapshots><enabled>false</enabled></snapshots>` (Chainguard has no snapshots); both `<repositories>` and `<pluginRepositories>` need it. For repo-manager mode, the built-in `central` is overridden with `<url>http://central</url>` (intentionally invalid) to force everything through the mirror. Maven build-output diagnostic: lines beginning `Downloaded from chainguard:` confirm Chainguard served the artifact; `Downloaded from central:` indicates fallback. In Gradle and Bazel, **the Chainguard repository must be listed before `mavenCentral()`** — the most common silent-fallback failure. Each Java artifact dir on `libraries.cgr.dev/java/<group>/<artifact>/<version>/` carries `.pom`, `.jar`, `.slsa-attestation.json`, and `.spdx.json`; curl needs `-L` to follow R2 redirects, `.netrc` works for `machine libraries.cgr.dev`.
- **JavaScript**: malware scanning via OSV/OpenSSF Malicious Packages — MAL-flagged packages permanently blocked, and detection is **continuous** (a version that worked yesterday can be added to the block list later). Layered on top is **Chainguard Sentinel scanning**, which blocks greyware and malicious packages **before a public advisory exists** — a Chainguard-specific control. The live block list is queryable via the **Malware Block List API** at `https://libraries.cgr.dev/javascript/-/api/malware` — see the dedicated section below for auth, query params, and the `source` field semantics. Per-package SPDX SBOMs available at `https://libraries.cgr.dev/javascript/-/npm/v1/sbom/spdx/<package>@<version>` (and `npm show <pkg>@<ver> dist.sboms` indicates which versions ship one). Env var idiom: `CHAINGUARD_JAVASCRIPT_IDENTITY_ID` / `CHAINGUARD_JAVASCRIPT_TOKEN`. **Artifactory redirect caching must be disabled** — `libraries.cgr.dev` 302-redirects to Cloudflare R2 (`9236a389bd48b984df91adc1bc924620.r2.cloudflarestorage.com`); Artifactory's default may cache the redirect URL instead of the blob. On the `javascript-chainguard` remote in Artifactory: enable **Bypass HEAD Requests**, disable **Lenient Host Authentication**, optionally enable **Cookie Management** (JFrog-recommended for redirect-based remotes), then **Zap Caches**. The Artifactory **Test button is unreliable** — it can fail for a correctly configured remote and pass for an incorrect one; validate by comparing checksums between a direct `curl` from `libraries.cgr.dev` and the same artifact fetched through Artifactory (`openssl dgst -sha512 -binary | base64`). When using a **two-remote GAR setup** (`javascript-chainguard` + `javascript-chainguard-upstream`), `artifactregistry-auth` only injects credentials for repos explicitly in `.npmrc` — add a credentials entry for **both** remotes or upstream-fallback packages return 404s. For private scoped packages, add `replace-registry-host=never` to `.npmrc`. For direct-access pnpm/npm against both Chainguard endpoints, write **two** `_auth` lines: `//libraries.cgr.dev/javascript/:_auth=…` and `//libraries.cgr.dev/javascript-upstream/:_auth=…` (the second is needed for upstream-fallback packages). Diagnostic for pnpm lockfiles: Chainguard-built entries carry only `resolution: {integrity: …}`, upstream-fallback entries carry an explicit `tarball:` URL pointing at `libraries.cgr.dev/javascript-upstream/`. Pull-token default TTL 30 days. **Bun** is officially supported (`bunfig.toml` `[install.registry]` with raw `username`/`password`, not base64). pnpm has three caches — after switching, clear all three to dodge stale-data 404s: `pnpm cache delete` (metadata), `rm -rf "${XDG_CACHE_HOME:-$HOME/.cache}/pnpm"` (HTTP), `rm -rf "$(pnpm store path)"` (content store). `pnpm prune` alone is **not** sufficient. For checksum-mismatch errors in Yarn Berry use `checksumBehavior: reset` in `.yarnrc.yml` instead of nuking the cache. Yarn Berry direct-access uses `.yarnrc.yml` `npmRegistries[...].npmAuthIdent` (raw `username:password`, not base64) plus `npmAlwaysAuth=true`.
- **Python**: Chainguard ships **manylinux** native binaries only (`manylinux_2_28` and `manylinux_2_39` variants — compatible with RHEL 8, Ubuntu 20.04, Amazon Linux 2023; glibc ≥ 2.28, x86_64/aarch64). Dev workflow on **Windows/macOS falls back to PyPI** for native-extension packages — configure a PyPI fallback or use WSL2/Docker. Four separate indexes (note the `/simple/` suffix in `pip`-style usage): `/python/simple/` (baseline), `/python-remediated/simple/` (CVE-patched packages tagged with a `+cgr.N` local-version suffix — Python resolvers treat `+cgr.N` as taking precedence over an unsuffixed version), and CUDA-specific `/cu126/simple/`, `/cu128/simple/`, `/cu129/simple/` (not dependency-complete — configure `https://pypi.nvidia.com` or PyPI as an additional source for toolkit components). For uv, set `index-strategy = unsafe-best-match` and `keyring-provider = "subprocess"` (uv disables keyring auth by default). Short-lived credentials via `pip install keyrings-chainguard-libraries` — re-install with `--ignore-installed --no-cache-dir` (pip) or `--reinstall --no-cache` (uv) to make sure you get Chainguard's copy of the keyring itself. Env vars: `CHAINGUARD_PYTHON_IDENTITY_ID` / `CHAINGUARD_PYTHON_TOKEN`. **Credential-escape rules differ by tool**: `uv.toml` wants the `/` in the username **percent-encoded** as `%2F`; `~/.pip/pip.conf` docs say to **replace `/` with `_`** literally. Beware **`uv sync --frozen`** — it bypasses index configuration entirely and downloads from URLs embedded in `uv.lock`; if that lockfile was generated against PyPI, `--frozen` keeps pulling from PyPI. Run `uv lock` (no `--frozen`) first, or `chainctl libraries update-hashes`. **`pip-compile` embeds credentials**: when given a credentialed index URL, it bakes `--index-url https://...:...@libraries.cgr.dev/...` into the generated `requirements.txt`. Strip the `--index-url` line before committing. **PEP 740 provenance** at `https://libraries.cgr.dev/python/integrity/<package>/<version>/<file>/provenance` (and `/bundle.json` for the Sigstore bundle); publisher identity is `chainguard-dev/ecosystems-wheel-rebuilder` running `python-build-versions.yaml`. **PEP 770 embedded SBOM** inside each wheel at `*.dist-info/sboms/sbom.spdx.json` with `creators` containing `Organization: Chainguard, Inc` — useful for in-place verification when network egress is restricted. **Python 3.9 EOL**: upstream EOL was Oct 31 2025; Chainguard's extended-support window ends **2026-05-15** — after that, no new 3.9 packages and no security updates on existing 3.9 packages. **Vendored-binary CVEs**: some Python packages bundle Go/Rust/C++; Chainguard may ship a `+cgr.N` version when a CVE exists only in the vendored dependency, with **no advisory in the VEX feed** — scanners that read advisories will miss it, scanners that inspect vendored binaries will see it. **Public VEX feed**: `https://libraries.cgr.dev/openvex/v1/all.json` (aggregate) and `https://libraries.cgr.dev/openvex/v1/pypi/<package>.openvex.json` (per-package). **pdm tooling note:** `update-hashes` supports `pdm.lock`, but Chainguard has not published a pdm-specific build-configuration guide — fall back to direct-access `.netrc` / env-var auth.
- **Go**: not supported by Chainguard Libraries — no `--repository=go` for pull tokens.

#### Scanner integration with Chainguard Libraries
- **Grype 0.100.0+** — source-directory scans (`grype .` against `requirements.txt`) report the declared version, not the installed one — `+cgr.N` is not recognized in this mode. Scan the installed venv (`grype venv`) or the container image instead.
- **Anchore Enterprise 5.23.0+** — **disable CPE matching** for the ecosystem in which Chainguard Libraries is used. Otherwise remediated CVEs are not filtered.
- **Trivy 0.54+** — requires the experimental VEX Repo feature. `trivy vex repo init` creates `~/.trivy/vex/repository.yaml`; add a `chainguard-libraries` entry at the top with `url: https://libraries.cgr.dev/openvex/v1`. Run `trivy filesystem . --vex repo --show-suppressed`. Gotcha: a plain `werkzeug==3.0.2` in `requirements.txt` (no `+cgr.N`) makes Trivy report a false positive — declare the local-version suffix explicitly.
- **Amazon Inspector** — supported for Python via Inspector's enhanced ECR scanning; no customer-side config.
- **Upwind** — supports CI/CD container scanning of Chainguard Libraries for Python (recognizes `+cgr.N` versions). Supported toolchains: uv/`pyproject.toml`, pip/`requirements.txt`, Poetry.

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

### `chainctl libraries policy` — library governance (chainctl ≥ v0.2.291)
Controls **which upstream packages your org can pull**. A *policy* configures automatic gates (cooldown, malware scanning) plus explicit block/allow lists; a *binding* activates a policy for an `(organization, ecosystem)` pair in `ENFORCE` or `DRY_RUN` mode. **Distinct from container-image `chainctl policies`** (that governs image pulls). Ecosystems are `JAVA`, `PYTHON`, `JAVASCRIPT`. A newly created policy is **inactive** until you `enable` it (or create a binding). `list` shows both `SYSTEM` and `CUSTOM` policies.

| Command | Description |
|---------|-------------|
| `create` | Create a custom library policy (inactive until enabled). |
| `update POLICY` | Update a policy. Block/allow lists are **replaced**, not merged. |
| `delete POLICY` | Delete a policy. |
| `describe POLICY` | Show the full policy incl. every block/allow entry. |
| `list` | List SYSTEM and CUSTOM policies. Flag: `--parent`. |
| `enable --policy POLICY` | Create/activate a binding. Default `--mode ENFORCE`. |
| `disable --policy POLICY` | Delete the binding(s). If `--mode` omitted, removes both ENFORCE and DRY_RUN bindings. |
| `binding create/delete/list/update` | Lower-level binding management (the libraries binding **has an `update`**, unlike the image-`policies` binding). |

**`create` / `update` flags:**
- `--name` — policy name (`create` only; `update` takes `POLICY` positionally).
- `--parent` — org to scope the policy to.
- `--description` — policy description.
- `--cooldown-days` (`int32`, default `-1`) — cooldown window in days: `0` disables, `1`–`30` explicit, omit (`-1`) to inherit the **7-day** system default.
- `--block` (`stringArray`, repeatable) — package to always deny, as `purl=<package-url>`; append `@<version>` to block a single version.
- `--allow` (`stringArray`, repeatable) — package permitted to bypass gates, comma-separated key=value: `purl=<package-url>[,bypass-cooldown=true][,bypass-malware=true][,justification="..."]`. **`justification` is required with `bypass-malware`.**

**purl format by ecosystem:** `pkg:pypi/<name>`, `pkg:npm/<name>`, `pkg:npm/%40<scope>/<name>` (scoped), `pkg:maven/<group>/<artifact>`. Append `@<version>` to scope to one version (e.g. `pkg:npm/lodash@4.17.20`).

**`enable` / `disable` / `binding create` flags:** `--policy`, `--parent`, `--ecosystem` (`JAVA|PYTHON|JAVASCRIPT`), `--mode` (`ENFORCE|DRY_RUN`, default `ENFORCE`). `binding delete BINDING_ID` and `binding update BINDING_ID --mode ...` take the binding ID positionally.

**Examples:**
```bash
# Custom 10-day JS cooldown policy, then activate it in enforce mode
chainctl libraries policy create --name=js-cooldown --cooldown-days=10
chainctl libraries policy enable --policy=js-cooldown --ecosystem=JAVASCRIPT --mode=ENFORCE

# Always block a known-bad version; confirm what's active
chainctl libraries policy update js-cooldown --block='purl=pkg:npm/lodash@4.17.20'
chainctl libraries policy binding list --parent=my-org
```

### `chainctl libraries packages blocked`
List packages withheld by an active Libraries policy. Defaults to `ENFORCE`-mode events from the **last 30 days**. (Parent group `chainctl libraries packages` — "Inspect Libraries packages".)

**Flags:** `--parent` (scope to an org), `--ecosystem` (`JAVA|PYTHON|JAVASCRIPT`), `--mode` (`ENFORCE|DRY_RUN`, default `ENFORCE`), `--package` (exact name match, case-insensitive).

```bash
# What did the policy block in JavaScript recently?
chainctl libraries packages blocked --parent=my-org --ecosystem=JAVASCRIPT

# Was a specific package blocked (incl. dry-run)?
chainctl libraries packages blocked --package=left-pad --mode=DRY_RUN
```

### JavaScript Malware Block List API (`/-/api/malware`)
`https://libraries.cgr.dev/javascript/-/api/malware` returns the JavaScript libraries currently **blocked due to malware or greyware**. These are detected by Chainguard's **Sentinel** scanning system (continuously monitors the public npm registry, scans source code with multiple signals/tools) plus all publicly reported malicious packages from **OSV MAL IDs**. (Distinct from the policy-driven `packages blocked` command above, which reports what *your org's* policy withheld; this API is the registry-wide malware block list.)

**Auth — two options:**
```bash
# Option 1: interactive / local — Bearer token
chainctl auth login
curl -H "Authorization: Bearer $(chainctl auth token --audience=libraries.cgr.dev)" \
  "https://libraries.cgr.dev/javascript/-/api/malware" | jq .

# Option 2: CI/CD — Basic auth from a pull-token .npmrc (long-lived, no browser; --ttl max 1 year)
chainctl auth configure-npm --pull-token --parent=<your-org>
AUTH=$(grep '_auth=' .npmrc | cut -d= -f2-)
curl -H "Authorization: Basic $AUTH" \
  "https://libraries.cgr.dev/javascript/-/api/malware" | jq .
```

**Query parameters:**
| Param | Description |
|-------|-------------|
| `scope` | `version`, `package` (some MAL IDs apply to the whole package, not one version), or `all` |
| `package` | Exact package name, e.g. `@antv/g` |
| `source` | Report source: `osv`, `chainguard`, `manual` |
| `since` | RFC3339 timestamp — entries blocked at/after this time (e.g. `2026-05-15T00:00:00Z`) |
| `page_size` | Results per page, `1`–`500` (default `100`) |
| `page_token` | Cursor from `next_page_token` in a previous response |

**`source` field semantics:**
- `manual` — Chainguard-detected block with **no advisory ID yet** (MAL ID omitted).
- `chainguard` — Chainguard-detected block **linked to a formal advisory** (has a MAL ID). When a `manual` block is later tied to an advisory, a **new `chainguard` entry is created** and the original `manual` entry remains.
- `osv` — sourced from the public OSV feed; **always has a MAL ID**.

A package can have entries from multiple sources sharing the same MAL ID (e.g. OSV blocks at package scope, then Chainguard enumerates individual versions). **Caveats:** the API does **not yet distinguish greyware from malware**, and the block **reason/justification for Chainguard-detected packages is not yet exposed** (coming soon).

```bash
# All blocked packages/versions since a date
curl -H "Authorization: Bearer $(chainctl auth token --audience=libraries.cgr.dev)" \
  "https://libraries.cgr.dev/javascript/-/api/malware?since=2026-05-15T00:00:00Z" | jq .
# Is a specific package blocked?
curl -H "Authorization: Bearer $(chainctl auth token --audience=libraries.cgr.dev)" \
  "https://libraries.cgr.dev/javascript/-/api/malware?package=@antv/g" | jq .
```

---

## agent — Agent Commands (The Guardener)

`chainctl agent`

The Guardener is an AI-powered migration agent that converts Dockerfiles to use Chainguard Containers. The agent runs in five named phases — **Parse → Translate → Build & Compare → Iterate → Validate** — and has access to: searching the Wolfi `APKINDEX`, finding which package provides a given binary or shared object, comparing installed packages and filesystem layers (the Build & Compare phase uses `syft` to generate SBOMs for both the original and migrated images), executing commands inside built images, and reading build-context files (`requirements.txt`, `package.json`, etc.) so it can make informed substitutions. The AI runs server-side; Docker builds and file access remain local.

**Note:** The Guardener is currently in beta. Your organization must join the waitlist at the Guardener landing page.

**Prerequisites:**
- `chainctl` installed and authenticated
- Docker installed and running locally
- **Two-tier role model:** `guardener.admin` (or `owner`) is required once per org to accept Guardener's legal terms; `guardener.user` is the minimum role for any subsequent user to run Guardener sessions. Get the org UIDP with `chainctl iam organizations list -o table`, then bind the appropriate role: `chainctl iam role-bindings create --parent <group-id> --identity <your-identity> --role guardener.user` (or `guardener.admin` for the user accepting terms).
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
- **Migration target** defaults to `cgr.dev/chainguard/wolfi-base:latest`. The `multi-stage` optimizer can switch the runtime stage to distroless `-dev` / runtime variants. Tested with Python, Go, Node.js, Java, Spring Boot (UBI-based), and multi-stage Argo CD builds.
- **Time to run**: 5–30+ minutes per migration; `optimize` runs **longer** than `build` because it does deeper analysis. Bump Bash timeout to 1800000 ms.
- **`--non-interactive`** doesn't just skip prompts — it **automatically selects the first suggestion** for every ambiguous decision. Set expectations: in non-interactive mode you're committing to the agent's first guess at each branch.
- **Don't upgrade language/app versions while migrating.** Match the Chainguard base version to whatever is already running; version-pump the app in a separate change.
- **Free tier exposes only `:latest` and `:latest-dev`.** Major.minor tags and `-slim` variants are paid Production Containers only.
- **Non-determinism**: AI-based — expect slightly different results across runs.
- **Session-end semantics**: a network interruption terminates the bidirectional gRPC stream and ends the session immediately. **`--resume` only resumes from locally saved migration state, not from the live session.** The server-side agent, its conversation history, and any in-flight work are lost when the connection drops. There is currently no server-side session recovery — don't retry mid-flight assuming the server is holding state.
- **No server-side builds**: Docker must run locally. Fully-managed headless mode is planned but not available.
- **`accept-terms`** is required on first use; the server prompts for it when missing. Run manually with `chainctl agent accept-terms --group <UIDP>` if you want to pre-accept.
- **`--group` and `--group-id`** are accepted as aliases for the same flag (Guardener prose says `--group-id`, every example uses `--group`). Value is the **org UIDP (40-char string)** — get it via `chainctl iam organizations list -o table` or the Console "Settings → Organization UID" field.
- **Output streams**: `build` prints its report to stdout; `optimize` and `upgrade` print reports to stderr; `validate` prints its report to stdout.
- **`--help` for `chainctl agent`** only shows `accept-terms` (only `accept-terms` is in the published reference), but `dockerfile` exists and works.

**Per-language migration gotchas** (useful when reviewing what the Guardener produced, or for manual migrations):
- **Node** — `node` user is UID `65532`, `WORKDIR=/app`, `NODE_PORT=3000`, `dumb-init` shipped. Docker Hub's `node` uses a wrapper entrypoint; Chainguard's does **not** — `docker run node /bin/sh` fails. Add `ENTRYPOINT ["/usr/bin/dumb-init", "--"]` for signal handling. **`*-slim` Node tags omit `npm` and `busybox`** — runtime stage only, never a builder.
- **Python** — entrypoint is `/usr/bin/python` for **both** `latest` and `latest-dev`. `docker run cgr.dev/chainguard/python echo hi` fails because `echo` becomes a python arg. If the Dockerfile prepends a venv to `PATH`, set an **explicit `ENTRYPOINT`** or system python (no venv packages) runs. Default user is `nonroot`; recommended `ENV PYTHONUNBUFFERED=1` and `ENV PYTHONDONTWRITEBYTECODE=1`.
- **.NET** — must `USER 0` before `dotnet restore` or any apk operation in the build stage. Runtime stages run non-root automatically — no `USER` directive needed (unlike Microsoft images which need `USER $APP_UID`). Use `cgr.dev/chainguard/aspnet-runtime:latest` for ASP.NET; `cgr.dev/chainguard/dotnet-runtime:latest` for .NET Core console apps.
- **Go** — the `static` runtime image has no glibc; the binary must be statically linked, so set `CGO_ENABLED=0` on the builder-stage `go build`. Otherwise use `glibc-dynamic`.
- **Java** — one-line rebase: `FROM maven` → `FROM cgr.dev/chainguard/maven`. Recommended pattern: `cgr.dev/chainguard/maven AS builder` → JRE image as the runner, copy the jar.
- **PHP** — extensions baked into all PHP images: `php-mbstring`, `-curl`, `-openssl`, `-iconv`, `-mysqlnd`, `-pdo`, `-pdo_sqlite`, `-pdo_mysql`, `-sodium`, `-phar`. Laravel image adds `-ctype`, `-dom`, `-fileinfo`, `-simplexml`. **Web apps use `latest-fpm` / `latest-fpm-dev`**, not plain `latest`. Laravel image has a `laravel` system user (UID 1000) for shared-volume dev. List enabled extensions: `docker run --rm --entrypoint php cgr.dev/chainguard/php -m`.

**Distro-compat & shell gotchas:**
- **Wolfi binaries are NOT Alpine binaries** — Wolfi is glibc, Alpine is musl. Don't copy Alpine binaries into a Wolfi base. Vendor-supplied binaries copied into Chainguard images must be glibc-built.
- **Package-manager mapping**: `apt install` → `apk add`, `apt remove` → `apk del`, `apt update` → `apk update` (Debian/Ubuntu). `yum install` → `apk add`, `yum remove` → `apk del`, `yum makecache` → `apk update` (Red Hat UBI). `dnf` and `microdnf` are accepted sources by `dfc`. Red Hat UBI has **no busybox by default** — change tooling expectations.
- **Wolfi busybox is slimmer than Alpine busybox** — some Alpine busybox utilities live in standalone packages in Wolfi (e.g., `groupadd` is in the `shadow` package).
- **APK search idioms**: `apk search cmd:<command>` finds the package providing a binary (the `cmd:` prefix also works directly with `apk add`); `apk -R info <pkg>` lists dependencies; `apk search so:<lib>.so*` finds packages providing a shared object.
- **Default shell is BusyBox `ash`** (not bash) on `-dev` and `chainguard-base`. Bash-isms (`echo {1..5}`, brace expansion, etc.) fail. Either rewrite for ash or `apk add bash`.
- **`-dev` images don't run as root** either, mostly. Common surprise: `docker run image-dev` hits permission errors. To get a root shell: `docker run --user root --entrypoint /bin/sh image-dev`.
- **Entrypoint inheritance**: many Docker Hub images wrap CMD via an entrypoint script. Chainguard images generally do not — read each per-image Specifications page before relying on bare-string `docker run image cmd` invocations.
- **PID-1 signal handling**: a Node process running as PID 1 doesn't handle `SIGTERM`, so `docker stop` waits and then `SIGKILL`s. Wrap with `dumb-init` (Node — already shipped) or `tini` (other runtimes) as the actual entrypoint.

**9-step migration checklist** (`migration-checklist`):
1. Check the per-image Containers Directory overview for usage details and compatibility remarks.
2. Replace your current base image with a standard `-dev` (such as `latest-dev`) variant as a starting point.
3. Add `USER root` before package-install or admin commands.
4. Replace `apt install` (or `yum install`) with `apk add`.
5. Use `apk search` on a running container (or the APK Explorer) to identify packages — names/bundling may differ.
6. Set proper file permissions when copying app files.
7. Switch back to a non-root user before finalizing the image.
8. Build and test your image to validate your setup.
9. **Optional:** migrate to a multi-stage build that uses a distroless image variant as runtime.

#### Dockerfile Converter (`dfc`) — non-AI alternative to the Guardener
`dfc` is a fast, **offline, rule-based** Dockerfile converter — the right tool when an LLM-driven iterative migration is overkill (quick first-pass conversion, predictable output). Supports Alpine (apk), Debian/Ubuntu (apt/apt-get), Fedora/RHEL/UBI (yum/dnf/microdnf).

Install: `brew install chainguard-dev/tap/dfc` or `go install github.com/chainguard-dev/dfc@latest`.

Key flags: `--org <orgname>`, `--registry <url>` (overrides `--org`), `--mappings <file.yaml>` (custom rules; docs use both `--mapping` and `--mappings` — `--mappings` matches the working example), `--in-place` (writes `.bak`), `--json` (programmatic output: top-level `lines[]` with per-line `raw`, `converted`, `stage`, and either `from.base` or `run.{distro, manager, packages[]}`). Default behavior is **stdout-only** — bare `dfc Dockerfile` prints to stdout and writes nothing on disk. Tag rules: semver truncated to major.minor; `-dev` added automatically **only when the stage has `RUN` commands** (so `FROM node` with no `RUN` produces `cgr.dev/<org>/node:latest`, no `-dev`); Debian/Ubuntu/RHEL bases always land on `chainguard-base:latest`. Also auto-rewrites `useradd` → `adduser` and `groupadd` → `addgroup` to match BusyBox syntax. Importable as a Go library: `github.com/chainguard-dev/dfc/pkg/dfc`. Also has an experimental **MCP server mode** for AI agents that want to call the rule engine as a tool.

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
# macOS (Homebrew) — Xcode CLT is a prerequisite (xcode-select --install)
brew tap chainguard-dev/tap
brew install chainctl

# Linux / macOS direct download
curl -o chainctl "https://dl.enforce.dev/chainctl/latest/chainctl_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/aarch64/arm64/')"
sudo install -o "$UID" -g "$(id -g)" -m 0755 chainctl /usr/local/bin/

# Windows (PowerShell, Admin or Windows Developer Mode for the symlink step)
curl -o chainctl.exe https://dl.enforce.dev/chainctl/latest/chainctl_windows_x86_64.exe
# add chainctl.exe's directory to $env:Path, then in that directory:
New-Item -ItemType SymbolicLink -Path "docker-credential-cgr.exe" -Target "chainctl.exe"
# Invoke as `.\chainctl`. Some commands are less tested on Windows.
```

**Cosign verification recipe (regulated customers should always do this — version-pinned form):**
```bash
PLATFORM=$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/aarch64/arm64/')
VERSION=$(curl -sS https://dl.enforce.dev/chainctl/latest/metadata.json | jq -r .version)
cosign verify-blob \
  --certificate chainctl.crt \
  --signature chainctl.sig \
  --certificate-identity "https://github.com/chainguard-dev/mono/.github/workflows/.release-drop.yaml@refs/tags/v${VERSION}" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  "$(which chainctl)"
# Expected output: "Verified OK"
```

**Cosign install** (since several recipes in this skill require it):
- `brew install cosign` (macOS / Linuxbrew) — `apk add cosign` (Alpine / Wolfi / from inside a Chainguard `-dev` image)
- `go install github.com/sigstore/cosign/v3/cmd/cosign@latest` (Go ≥ 1.22.7)
- Direct binary: `https://github.com/sigstore/cosign/releases/latest/download/cosign-<os>-<arch>`

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

**TLS requirements:** minimum TLSv1.3 with `TLS_AES_256_GCM_SHA384`, or TLSv1.2 with `ECDHE-{ECDSA,RSA}-AES256-GCM-SHA384`, RFC 7627 Extended Master Secret, P-256/SHA-256 or RSA-2048/SHA-256. Some legacy FIPS 140-2 middleware will fail. FIPS containers require TLS 1.3.

Test a server's compatibility:
```bash
openssl s_client -cipher @SECLEVEL=2:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384 \
  -ciphersuites TLS_AES_256_GCM_SHA384 -groups P-256 -connect HOST:PORT
# For TLSv1.2 the output must include "Extended master secret: yes".
```

**Known-incompatible TLS clients** (use the FIPS-image workaround instead): **AWS MQ RabbitMQ** (no TLSv1.2 RFC 7627 or TLSv1.3) → use Chainguard `rabbitmq-fips`. **AWS RDS Postgres < 16.11-R1** → use Chainguard `postgres-fips`. **Amazon Linux 2** default OpenSSL has no approved mode → add `openssl11`, migrate to AL2023/Bottlerocket, or use a Chainguard VM (AL2 sunsets 2026-06-30).

---

## Platform API

The Chainguard platform exposes v1 and v2beta1 surfaces at `https://console-api.enforce.dev/`. `chainctl auth token` is the standard way to obtain a bearer token; pair with `curl` or any HTTP client.

```bash
curl -H "Authorization: Bearer $(chainctl auth token)" \
  https://console-api.enforce.dev/iam/v1/groups | jq .
```

**Sample v1 endpoints:**
- `GET /iam/v1/groups` — list orgs/folders you're a member of.
- `GET /registry/v1/repos` — list repos.
- `GET /registry/v1/eoltags?uidp.childrenOf=<REPO_UIDP>` — EOL/grace status (see Image lifecycle below).

**Tag History API** (registry-served, the HTTP form of `chainctl images history`):
```bash
curl -H "Authorization: Bearer $(chainctl auth token)" \
  https://cgr.dev/v2/<ORG>/<IMAGE>/_chainguard/history/<TAG> | jq .
# public images: <ORG> is always "chainguard"
```
Returns the digest history for a tag. **Max 1000 records per request** — for tags with many digests, page with `start`/`end` ISO-8601 timestamp query params.

**API v2 (v2beta1)** is in limited beta and follows AIP conventions:
- Path prefixes `/iam/v2beta1/`, `/registry/v2beta1/`, `/vulnerabilities/v2beta1/`.
- Cursor pagination: `page_size` (default 50, max 200), `page_token`, `order_by` (e.g. `created_at desc` or `createdAt desc` — both snake_case and camelCase accepted; URL-encode the space as `%20`), `skip=<n>` for random-access pagination (response includes `skipped` field confirming how many were skipped). Tokens expire after 3 days (AIP-158).
- Response field renames vs. v1: `id` → `uid`, `createdAt` → `createTime`, `updatedAt` → `updateTime`. Responses also carry `nextPageToken`, `totalCount`, `skipped`.
- **Hydrated references** — role-binding responses include full identity, group, and role objects, so v2 callers don't need follow-up lookups that v1 callers do.
- **Direct UID `Get`** — `GET /iam/v2beta1/groups/<uid>` returns the resource directly (in v1 this required a List with an `id` filter).
- Parent goes in the URL path on POST: `POST /iam/v2beta1/groups/$ORG_ID` with body `{"name":"...","description":"..."}`.
- `PATCH ...?updateMask=<field>` does partial updates; **`updateMask` is optional** in v2 — without it, only fields present in the body change (unlike v1 PATCH, which often required full-resource replacement).
- Structured errors follow AIP-193 with `details[]` of typed payloads (`google.rpc.ErrorInfo`, `google.rpc.BadRequest` with `fieldViolations`, `google.rpc.PreconditionFailure`).
- **Rate limits are not enforced during beta**; they will be introduced at GA.
- gRPC is available at the same host; protos at `chainguard.dev/sdk/proto/chainguard/platform/`.

**Token-exchange endpoint** — direct OIDC → API token swap (useful in CI containers that don't have `chainctl` installed):
```bash
API_TOKEN=$(curl -sSf \
  -H "Authorization: Bearer $IDENTITY_TOKEN" \
  "https://issuer.enforce.dev/sts/exchange?aud=https://console-api.enforce.dev&identity=$IDENTITY" \
  | jq -r .token)
```
Token TTL is 1 hour. `aud` is configurable; `identity` is the UIDP of the assumable identity to mint a token for.

**Programmatic Go clients:** `chainguard.dev/sdk/auth` provides `auth.NewChainctlTokenSource(ctx, auth.WithAudience("cgr.dev"))`; `chainguard.dev/sdk/auth/ggcr` provides a `go-containerregistry` keychain (`ggcr.TokenSourceKeychain`). `ggcr.Keychain(sub, ts)` supports impersonating an assumable identity (`sub` = identity UIDP).

**OpenAPI specs:** `/chainguard/api/spec/` (unified), `/chainguard/api/spec-api-v1/`, `/chainguard/api/spec-api-v2/`.

---

## Image Matcher — find the closest Chainguard image (API)

An API-based migration helper that analyzes a **source image's SBOM** and returns a ranked list of the closest-matching Chainguard images. **Deterministic** (curated equivalences + CPE/PURL matching + runtime-module overlap — no ML). Not a `chainctl` subcommand; call it directly.

- **Endpoint:** `POST https://console-api.enforce.dev/image-recommendation/v1/match`, auth `Authorization: Bearer $(chainctl auth token)`.
- **Request body:** `parent_id` (org UID), `dist_name`, `arch`, optional `dist_version`, and `sbom.sbomComponents[]` (each: `type`, `purl`, `name`, `cpe`, `bomref`, `version`). Input must be **CycloneDX JSON** with PURLs; **rename `bom-ref` → `bomref`**.
- **Supported `dist_name` / purl prefix:** `debian`/`ubuntu` (`pkg:deb/`), `alpine` (`pkg:apk/`), `redhat`/`suse`/`amazon-linux` (`pkg:rpm/`).
- **Response:** `images[]` ranked by `probabilityScore` (0–100; **≥90 strong, 50–70 review**), each with `imageRef`, `satisfiedCount`, `totalRequired`, `coverage`, `satisfiedPackages`, `missingPackages`, `extraPackages`; plus top-level `totalExternalPackages`, `requiredApks`, `unmatchedExternalPkgs`, and an `overallScore` to break ties among 100-capped scores. Defaults: up to **10** candidates (`count`), `threshold` **50.0**. The primary package is weighted far more heavily than the rest.

## Octo STS — short-lived GitHub tokens via OIDC

[Octo STS](https://octo-sts.dev) is a Chainguard-built GitHub App (`https://github.com/apps/octo-sts`) that exchanges any OIDC token for a short-lived **(1-hour, non-refreshable)** GitHub access token. Trust policies live in the repo at `.github/chainguard/<name>.sts.yaml`. Eliminates PATs for Renovate, cross-repo automation, org-level workflows, and external services pushing to GitHub. Hosted free at `octo-sts.dev`; self-hostable.

Minimal trust policy:
```yaml
# .github/chainguard/renovate.sts.yaml
issuer: https://token.actions.githubusercontent.com
subject: repo:org/repo:ref:refs/heads/main
# subject_pattern: for regex (prefer exact `subject:` when possible — narrower trust surface)
# repositories: [org/repo-one, org/repo-two]  # top-level sibling of permissions, multi-repo from one policy
permissions:
  contents: read
```
The filename (here `renovate`) is the `identity` slug used at exchange time. **Branch protection still applies** — even with `contents: write`, GitHub-side branch protection (required PRs, status checks) is enforced; Octo STS doesn't let you push directly to protected branches.

Exchange URL pattern:
```
GET https://octo-sts.dev/sts/exchange?scope=<org>/<repo>&identity=<policy-name>
Authorization: Bearer <OIDC_TOKEN>
→ {"access_token": "..."}
```

**Relationship to chainctl identities — they are complementary, not duplicates.** A chainctl assumable identity lets a GitHub Actions job authenticate to `cgr.dev`. Octo STS lets a workflow that already authenticated via OIDC (including one that just authed to Chainguard) push back to GitHub without a PAT. The Renovate-with-Chainguard pattern chains them: first `chainctl auth login --identity ...` for registry access, then Octo STS for the GitHub push token (the GitHub `GITHUB_TOKEN` can't update workflow files; an Octo STS token can).

---

## Image lifecycle (EOL / grace period)

- Chainguard publishes **EOL grace periods up to 6 months** after upstream EOL. The 6-month limit is hard — **no exceptions**. The grace period ends **immediately if a build fails due to dependency conflict**.
- **Eligibility criteria for a grace period:** (1) primary package is listed as a current or EOL version of a version-stream package in the catalog; (2) has multiple release tracks; (3) within 6 months of the official upstream EOL; (4) release/EOL dates are available on endoflife.date.
- **Grace does NOT cover**: updating the EOL primary package itself, backporting/cherry-picking patches to it, or packages already EOL for more than 6 months.
- **When a grace period ends**, the org keeps access to the **last successful build** — the image becomes static and stops receiving CVE remediations.
- `chainctl packages versions list <pkg> --include-inactive` shows packages in the grace window.
- **Chainguard OS has no LTS / pinned-version catalog**: every available package is always installable, and there's no "RHEL 8"-style frozen distribution. EOL grace is the mechanism for staying on a specific major version while you migrate.
- Listing EOL tags programmatically — the `REPO_UIDP` comes from the `ID` column of `chainctl images repos list --parent <org> -o wide` (e.g., `ORGANIZATION_ID/4408EXAMPLE4131a`):
  ```bash
  REPO_UIDP=$(chainctl images repos list --parent my-org -o wide | awk '/node/ {print $1; exit}')
  curl -s -H "Authorization: Bearer $(chainctl auth token)" \
    "https://console-api.enforce.dev/registry/v1/eoltags?uidp.childrenOf=${REPO_UIDP}" | jq .
  ```
- API returns: `tagStatus` ∈ {`TAG_ACTIVE`, `TAG_IN_GRACE`, `TAG_INACTIVE`}; `graceStatus` ∈ {`GRACE_ACTIVE`, `GRACE_ELIGIBLE`, `GRACE_NOT_ELIGIBLE`}; plus `mainPackageName`, `mainPackageVersion.{eolDate, lts, releaseDate, version, fips, eolBroken}`, and `gracePeriodExpiryDate` (ISO 8601). Newer doc examples also expose `legacyLts` (string), `latestVersion` (string), and `versionSource` (often null). **`lts` is polymorphic** — older API responses show a date string (`"lts": "2022-10-25"`), newer ones show a boolean (`"lts": false`). Parsers should accept both.

---

## Compliance — FIPS / STIG / FedRAMP / SLSA

- **Ask the user about compliance level** early — it affects image variant selection and TLS posture.
- FIPS image naming: `-fips` suffix. The catalog spans **700+** FIPS variants (browse `images.chainguard.dev/?category=fips`): `python-fips`, `nginx-fips`, `jdk-fips`, `jre-fips`, `go-fips`, `go-msft-fips`, `go-openssl`, `node-fips`, `dotnet-runtime-fips`, `postgres-fips`, `rabbitmq-fips`, `busybox-fips`, `envoy-fips`, `gradle-fips`, `glibc-openssl-fips`, `spark-fips`, `spark-operator-fips`, and more. **FIPS containers are paid-tier only** — not in the free public mirror.
- **`spark-fips`** is notable: Apache Spark upstream has no FIPS crypto support, so Chainguard's is billed as "the first FIPS-validated image for Apache Spark." It ships **both** Bouncy Castle FIPS (140-3) **and** a validated redistribution of OpenSSL's FIPS provider. (`spark-operator-fips` is the operator variant.)
- `-fips-mip` variants ship Modules-In-Process whose CMVP validation is still pending; SBOMs tag them with `NIST-MIP-<module-name>`.
- **STIG-hardened by default**: all Chainguard FIPS Containers include STIG hardening (DISA "General Purpose Operating System SRG" profile). There is **one** Chainguard STIG that applies to all containers — there are no per-image STIGs (mysql STIG, etc.) from Chainguard.
- FIPS images use **kernel-independent** Jitterentropy userspace entropy — run on any recent Linux kernel (including dev laptops). Caveats: some CNI plugins, LUKS2, StrongSwan still need FIPS-mode kernel. Kernel-independent FIPS requires `libcrypto3 >= 3.4.0-r2` and `openssl-config-fipshardened >= 3.4.0-r3`, on images built **from 2024-11-07 onward**. **Excluded historically**: `bouncycastle-fips`, `bouncycastle-fips-1.0`, any package with `-cni-` in the name. **As of August 2025**, Java FIPS images additionally support kernel-independent operation via a "Bouncy Castle Entropy Provider with Jitterentropy" integration — so the `bouncycastle-fips` exclusion no longer applies unconditionally; check the per-image attestation when assessing FIPS-on-non-FIPS-kernel suitability.
- **Documented runtime support matrix** for kernel-independent FIPS: Go (1.21+), Python (3.10–3.13), Node.js (18–23), .NET (6–8), PHP (8.2–8.3), C/C++, Java (11, 17, 21, as of Aug 2025).
- **CMVP cert #4282** carries a caveat requiring "an approved entropy source" — the matched ESV (entropy source validation) certificate satisfies that caveat. Always pair the two when citing the FIPS module.
- **FIPS 140-2 → 140-3 deadline:** "As of September 2026, all FIPS 140-2 certificates will become historical, making migration to FIPS 140-3 essential." Plan migrations accordingly.
- **FIPS 186-5 / EdDSA (Ed25519, Ed448):** FIPS 186-5 (published 2023) approves EdDSA for digital signatures — relevant when customers ask whether modern signature algorithms are FIPS-permitted.
- **Three crypto module families** ship across the catalog (don't assume OpenSSL everywhere):
  - **OpenSSL FIPS provider** (CMVP #4282 / #4985) — most images.
  - **Bouncy Castle FIPS** (BCFIPS / BC-FJA, v2.1.1) — Java FIPS images.
  - **BoringCrypto** (BoringSSL-FIPS-2023042800) — `envoy-fips` and forks. Plain `envoy` uses non-FIPS BoringSSL and is NOT FIPS-validated.
- **SBOM evidence**: packages with `NIST-CMVP-<n>` (validation cert), `NIST-ESV-<n>` (entropy-source cert), and `NIST-MIP-<module>` (modules-in-process) prefixes encode the certificate numbers — useful for auditors. Quick SBOM grab: `cosign download sbom cgr.dev/$ORG/$IMAGE:$TAG`.
- **Built-in verification commands**: every FIPS image ships `openssl-fips-test`:
  ```bash
  docker run -it --entrypoint openssl-fips-test cgr.dev/$ORG/python-fips
  # prints public API version, FIPS module version, self-test results, CMVP search URL
  ```
  Java FIPS: `docker run -it --rm cgr.dev/$ORG/jre-fips org.bouncycastle.util.DumpInfo` (reports FIPS Ready / Native Ready status). `envoy-fips`: `bssl-test_fips` binary, plus `envoy-fips --version` reports `BoringSSL-FIPS-2023042800`.
- **Approved-only mode is non-toggleable.** To use non-FIPS algorithms in a FIPS image, switch to the non-FIPS image. Non-security MD5/SHA-1 can be opted in per-call (OpenSSL property query `?fips=yes` / `-fips`, or Python `hashlib.md5(data, usedforsecurity=False)`); HMAC, signing, and KDF remain blocked. SHA-1 was approved through OpenSSL FIPS provider v3.4.0, non-approved starting v3.6.0.
- **No FIPS support for** Rust (ring/aws-lc-rs double-linking issues), Mozilla NSS-based stacks, WireGuard / Tailscale / Shadowsocks. For tunnels, OpenVPN or StrongSwan are the FIPS-compatible options.
- **`GOEXPERIMENT=boringcrypto`** upstream Go is NOT covered by the Chainguard FIPS Commitment — use `go-fips` / `go-msft-fips` / `go-openssl` instead. Different MD5 semantics: `go-fips` allows MD5, `go-msft-fips` blocks it.
- **FIPS Commitment does not apply to Custom Assembly outputs**, even when the base is a `-fips` image.
- **FIPS EKS Add-ons** (zero-CVE, FIPS 140-3 validated, distributed via AWS Marketplace — no Chainguard account required, billed via AWS): `kube-proxy`, `CoreDNS`, `Amazon VPC CNI`, `Amazon EBS CSI Driver`, `Amazon EFS CSI Driver`.
- **FedRAMP Rev 5 / SC-8 scope.** "Any data in transit, whether from one container to another or to a sidecar inside the same host virtual machine" requires FIPS-validated encryption controls under SC-8. Useful to cite when explaining why a service mesh or cluster-internal TLS still needs FIPS modules — not just north-south traffic.
- **STIG scanning** of a Chainguard image: use the `openscap` Chainguard image with the `ssg-chainguard-gpos-ds.xml` datastream. Patterns:
  ```bash
  # scan a registry image (requires local docker pull first)
  oscap-docker image cgr.dev/chainguard/nginx:latest xccdf eval --datastream-id ssg-chainguard-ds.xml ...
  # scan a running container
  oscap-docker container <container-name> xccdf eval ...
  # both run against cgr.dev/chainguard/openscap:latest-dev
  ```
- **Chainguard VMs** ship Secure Boot, **hybrid CIS Level 1 + STIG baseline** (combining CIS Level 1 with STIG controls), FIPS 140-3 validated modules + SP 800-90B entropy, and the same 7-day-critical / 14-day-high CVE SLA as containers. **Node-replacement upgrades** only (no in-place). Compliance artifacts include POA&M, SCAP/OpenSCAP STIG+CIS scan results, and OpenSSL FIPS docs. VMs are **not** managed by `chainctl` — distributed via GCP Compute Engine, AWS (EC2/ECS/EKS), Azure Compute, QEMU/KVM (qcow2 and raw), VMware vSphere (VMDK), Nutanix (qcow2/raw). VM kernel posture: "Chainguard Factory tracks both the stable upstream and the latest LTS (for FIPS) versions of the kernel, building from source." Three named VM tiers: **Container Host VMs** (run containers), **Base VMs** (Chainguard Base, Java Base, Python Base — build your own), **Application VMs** (Nginx, Jenkins, Squid Proxy — turnkey). **Kernel-independent FIPS applies to VMs too** — application workloads use a FIPS-validated entropy source independent of the kernel, so VMs no longer need to be booted in FIPS mode. Caveat: low-level functions (disk encryption, IPsec, KMSV) still don't use FIPS-validated entropy with kernel-independent mode; on cloud platforms disk/network encryption is handled by the cloud provider's own FIPS-validated entropy.
- **Chainguard OS vs Wolfi**: **Wolfi** is the OS under free-tier Chainguard Containers (`cgr.dev/chainguard/*`, mirrored to Docker Hub; APKs at `packages.wolfi.dev`). **Chainguard OS** is the production distribution under paid Chainguard products (private images, VMs, Libraries), with APKs at `apk.cgr.dev` / `packages.cgr.dev`. **Mixing content across Wolfi and Chainguard OS is explicitly unsupported.** A separate beta product, **Chainguard OS Packages**, exposes the full ~30,000-package Chainguard catalog for customers building their own images with Bazel/Dockerfiles/`rules_apko` — not compatible with Custom Assembly.
- **"Chainguard Repository"** is the named product for unified, policy-governed library access. Currently scoped to Chainguard Libraries for JavaScript at `libraries.cgr.dev/javascript`. Don't confuse with `chainctl images repos` (image repos) or APK repos.
- **CIS Docker Benchmark conformance**: Section 4 generally conformant with two caveats — **4.5 Content Trust** uses Cosign instead of Docker Content Trust (auditor discretion); **4.6 HEALTHCHECK** is not provided (Kubernetes does this at the orchestration layer, not Docker) — does not meet the benchmark literally.
- **SLSA Level 3** applies to all Chainguard products (Containers, VMs, Libraries). Note the distinction in attestations below: the **predicate** is `https://slsa.dev/provenance/v1` — that's the provenance-format version, **not** SLSA Level 1. Implementation specifics: each build runs in a dedicated ephemeral environment, provenance is signed by the build-system control plane (not workers), signing keys never reside on build workers (off-box signing service), SBOM + provenance per artifact. Builder ID: `https://chainguard.dev/prod/builders/apkoaas/v1`; buildType: `https://chainguard.dev/buildtypes/apkoaas/v1` — useful when writing cosign policies.
- **PCI DSS 4.0**: Chainguard maps to requirements **6.3.1, 6.3.2** (identify/rank vulnerabilities; patch within timeframes) and **11.3.1, 11.3.11, 11.4.4** (vulnerability scanning/remediation) — low-CVE images plus the CVE-remediation SLA are the supporting controls.
- **CMMC 2.0**: targets **Levels 2 and 3**; relevant practices include **CM.2.062** (least functionality), **CM.3.068** (restrict nonessential software), and **SC.3.177** (FIPS-validated cryptography — satisfied by Chainguard FIPS Containers).

---

## Attestations & signatures — cosign workflow

chainctl does not itself verify signatures; use `cosign` alongside. **Fulcio** is the Sigstore CA — it issues ~20-minute ephemeral x509 certs based on an OIDC token. **Rekor** is the transparency log that stores the entries. That's why keyless verify needs no public key — `--certificate-identity` (the OIDC subject) plus `--certificate-oidc-issuer` (the OIDC issuer URL) is sufficient.

**SBOM vs attestation**: an SBOM is a list of components; an attestation is a signed in-toto envelope wrapping a predicate (SBOM, provenance, VEX, etc.) with verifiable provenance. **`cosign attach sbom`** uploads an *unsigned* SBOM as an OCI artifact with the `.sbom` tag; **`cosign attest --predicate file --type spdxjson`** generates a *signed* attestation with the `.att` tag; **`cosign verify-attestation`** verifies the signed one. Chainguard publishes signed attestations — `verify-attestation`, not `download sbom`. **`cosign clean <image>`** removes signatures/attestations/SBOMs from a registry — needed when re-attesting an image you've added packages to.

Every Chainguard image has:

| Predicate type | Cosign `--type` shorthand | Purpose |
|----------------|----------------------------|---------|
| `https://slsa.dev/provenance/v1` | `slsaprovenance` | SLSA build provenance (format version v1; product compliance is **SLSA Level 3** — different concept) |
| `https://spdx.dev/Document` | `spdxjson` | SPDX SBOM |
| `https://cyclonedx.org/bom` | `cyclonedx` | CycloneDX SBOM (customer-only, builds ≥ 2026-01-29) |
| `https://apko.dev/image-configuration` | — | apko build configuration (matches the YAML used at build time — `apko` is the declarative single-layer image builder that produced every Chainguard image; `melange` builds the apks that apko composes) |
| `https://chainguard.dev/end-of-life` | — | EOL metadata |
| `https://chainguard.dev/helm-values/v1` | — | Helm values attestation |
| `https://chainguard.dev/attestation/chart-lock/v1` | — | Helm chart-lock |
| `https://chainguard.dev/attestation/syft/v1` | — | Syft SBOM |
| `https://openvex.dev/ns/v0.2.0` | — | OpenVEX vulnerability statements |
| (signed vulnerability scan) | `vuln` | Sigstore vulnerability attestation |
| (generic in-toto) | `custom` | Custom predicates |

**OpenVEX** is a JSON-LD format for VEX (Vulnerability Exploitability eXchange) statements. Statuses: `not_affected`, `affected`, `fixed`, `under_investigation`. `not_affected` requires a `justification` (e.g. `component_not_present`, `inline_mitigations_already_exist`) or `impact_statement`. Chainguard publishes per-package VEX data so scanners that consume it (Grype, Trivy with VEX repo) can suppress false positives. Author with `vexctl`:
```bash
vexctl attest --attach --sign hello.vex.json <image>
# produces a .att attestation; verify with cosign verify-attestation --type https://openvex.dev/ns
```

**Keyless verify pattern** — `--certificate-identity[-regexp]` plus `--certificate-oidc-issuer`. Common issuer values: `https://token.actions.githubusercontent.com` (GitHub Actions), `https://accounts.google.com` (Google), `https://github.com/login/oauth` (interactive GitHub user login). Identity values for workflow signers look like `https://github.com/<org>/<repo>/.github/workflows/<file>.yml@refs/heads/main`. Custom Assembly outputs are signed by per-org `CATALOG_SYNCER` and `APKO_BUILDER` identities — different from the `chainguard-dev/mono` regex that signs public images.

**Verify any keyless-signed release** (works for cosign, crane, melange, apko, chainctl, and most Chainguard-published binaries):
```bash
cosign verify-blob \
  --signature <release.tar.gz.sig> \
  --certificate <release.tar.gz.crt> \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  --certificate-identity "https://github.com/<org>/<repo>/.github/workflows/release.yaml@refs/heads/main" \
  <release.tar.gz>
```

Example attestation reads:
```bash
cosign verify-attestation --type https://slsa.dev/provenance/v1 cgr.dev/$ORG/nginx:1.25
cosign download attestation --predicate-type https://spdx.dev/Document cgr.dev/$ORG/nginx:1.25 | jq -r .payload | base64 -d | jq .
# Multi-arch images need an explicit platform:
cosign download attestation --predicate-type https://spdx.dev/Document --platform=linux/amd64 cgr.dev/$ORG/nginx:1.25
```

**Two SBOM retrieval paths** in Chainguard images: (a) Sigstore attestation on the registry (cosign command above) and (b) embedded inside the image filesystem at `/var/lib/db/sbom/` (one SPDX JSON per package). For minimal images without a shell, `docker cp` the path out or use `crane export <image> - | tar -xOf - var/lib/db/sbom/<pkg>.spdx.json`.

**Signing your own images in GitHub Actions** (for customers re-signing Chainguard bases after their own build steps), the workflow needs:
```yaml
permissions:
  contents: read
  packages: write
  id-token: write    # required by cosign keyless / Fulcio
steps:
  - uses: sigstore/cosign-installer@main
    with:
      cosign-release: 'v3.0.2'
```
The `id-token: write` permission cannot be granted to PR workflows on forks.

Referrer tags (`sha256-<digest>.sig|sbom|att`) are exposed via `chainctl images list --show-referrers`.

---

## Troubleshooting

1. **Verbose output for bug reports:** `chainctl --v 5 <cmd> 2>&1 > run.log`. Also attach `chainctl version` and `chainctl config view` (scrub secrets). Alternatively use `--log-level debug`.
2. **Force re-authentication:** delete the token cache — `rm -rf ~/.cache/chainguard/<audience>/` (Linux) or `~/Library/Caches/chainguard/<audience>/` (macOS). The audience's `/` is replaced with `-` on disk.
3. **401s in long-running CI**: check whether the pull token has expired (default 30 days). Tokens rotate via refresh-token — if refresh failed, re-run `configure-docker --pull-token --save`.
4. **Stuck Guardener migration**: use `--log-file <path>` to capture detailed logs; extend Bash timeout to 1800000 ms or longer.
5. **`libraries verify` returns 0% on fat JARs**: expected — verify individual JARs from `~/.m2/repository` before assembly.
6. **Custom Assembly build timeout (~1 hour)**: normal builds complete in <20 min. Use `chainctl images repos build logs --repo=<name> --build-id=<id>` to read server-side logs.
7. **npm 404 on a brand-new upstream version**: expected during the 7-day cooldown on `CHAINGUARD_AND_UPSTREAM`. With upstream fallback enabled, the cooldown also applies to brand-new Chainguard-built versions. Retry after the window, lower the cooldown via `chainctl libraries policy create/update --cooldown-days` (range `0`–`30`; `0` disables — note this moved off `entitlements create` in chainctl `v0.2.291`), bypass it for a specific package with a policy `--allow` entry (`bypass-cooldown=true`), or pin a slightly older version. Config changes take up to 30 minutes to apply.
8. **Stale `chainctl update` cache**: `rm -f ~/.cache/chainctl/chainctl.bak`.
9. **Java build looks fine but everything came from Maven Central** — Gradle repository order matters. The Chainguard `maven { url = ... }` block must be **above** `mavenCentral()`. Same for Bazel `MODULE.bazel` `repositories = [...]`. Maven build log signal: `Downloaded from chainguard:` ✓ vs. `Downloaded from central:` ✗.
10. **`uv sync --frozen` pulls from PyPI despite Chainguard config** — `--frozen` bypasses index configuration entirely and uses URLs embedded in `uv.lock`. Run `uv lock` (no `--frozen`) first, or `chainctl libraries update-hashes uv.lock`.
11. **pnpm stale-data 404 after switching to Chainguard** — clear all three caches (metadata via `pnpm cache delete`, HTTP via `rm -rf "${XDG_CACHE_HOME:-$HOME/.cache}/pnpm"`, content store via `rm -rf "$(pnpm store path)"`). `pnpm prune` alone won't do it.
12. **Artifactory checksum mismatch via `libraries.cgr.dev`** — Artifactory caches the 302 redirect target instead of the blob. Validate by `curl`ing directly vs. through Artifactory and comparing hashes (JS: `openssl dgst -sha512 -binary | base64`; Java/Python: `sha256sum`). Fix on the `javascript-chainguard` remote: enable Bypass HEAD Requests, disable Lenient Host Authentication, Zap Caches.
13. **Custom Assembly `repo.create` error** — only the built-in `owner` role has both `repo.update` and `repo.create` by default. Either bind your identity to `owner` or build a custom role with both capabilities.
14. **Pull tokens "rotated and now fail"** — `configure-docker --pull-token --save` overwrites any prior credential and the old token cannot be recovered. Always extract the previous token (or run with `--save` only once per system) before re-saving.
15. **Notifications: Slack channel doesn't appear in the dropdown** — two common causes: (a) the user signed into the Chainguard Console with a different email than their Slack identity → disconnect, sign in with the matching email, reconnect; (b) the channel is **private** and the Chainguard Notifications app hasn't been added inside the channel yet → in Slack, Channel → Integrations → Add an App → Chainguard Notifications. Also: only `owner` can configure notifications.

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

# Create a new variant of an image (edit, not apply — --save-as is edit-only)
chainctl images repos build edit --repo=python --file=config.yaml --save-as=my-custom-python --parent my-org

# Apply config from file to an EXISTING repo (CI/CD)
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
The repo URL inside the container (`cat /etc/apk/repositories`) is `https://apk.cgr.dev/<long-hex-id>` — the hex value is the org's UIDP, mapped to a name via `chainctl iam organizations ls -o table`. The `HTTP_AUTH` env-var format is literally `basic:<host>:<user>:<password>` where `<user>` is the placeholder `user` and `<password>` is the ephemeral token. Catalog customers can opt into a beta that extends the private APK repo to the **full ~30,000-package Chainguard OS / Wolfi catalog**.

### Kubernetes `imagePullSecrets` from a pull token
```bash
chainctl auth configure-docker --pull-token --save --parent $ORG --ttl 720h
kubectl create secret generic regcred \
  --from-file=.dockerconfigjson="$HOME/.docker/config.json" \
  --type=kubernetes.io/dockerconfigjson
# reference under Pod.spec.imagePullSecrets[].name = regcred
```
Warning: this secret contains **all** Docker creds in `~/.docker/config.json` — extract a `cgr.dev`-only file if other registries are present.

### Cursor IDE integration
Prereqs: `chainctl` installed and authenticated; pull-token credentials injected via env vars for whichever ecosystems you're using. Then in a Cursor chat:
> I'd like to migrate this project to use Chainguard images and libraries. My Chainguard org is `<your-org>`.

Expected output: multi-stage Dockerfile rebased on `cgr.dev/chainguard/<lang>:latest-dev` (builder) and `cgr.dev/chainguard/<lang>:latest` (runtime), plus per-ecosystem config (Maven `settings.xml`, `.npmrc`, pip config, etc.). Common failure modes: 401/403 on install (verify pull-token ecosystem + 30-day TTL); image tag not found (use `latest`/`latest-dev` on free tier); package not found (configure upstream fallback per ecosystem — `mavenCentral()` for Java, `extra-index-url = https://pypi.org/simple/` for Python).

### Kiro IDE integration ("Chainguard Power for Kiro")
Distributed as a **Power** (not manual MCP config). Install via Kiro IDE → **Powers panel** in the sidebar → scroll to **Chainguard Power** → Install. Kiro auto-registers the MCP servers — **no JSON editing or separate CLI setup** for the Power itself. It bundles **four MCP servers**: `cg-api` (platform/org workflows), `cg-apk` (Wolfi package discovery), `cg-oci` (container image discovery + tag lookup), `cg-versions` (version & upgrade-path lookup). It reuses your **existing `chainctl` session** for image/package lookups, so `chainctl` must still be installed and authenticated first.

**Prereqs:** Kiro account + IDE; Chainguard org in **domain format** (e.g. `acme-corp.com`); access to Containers (image migration) and/or Libraries (Java/JS/Python dependency migration); a Libraries **entitlement** if using Libraries. **Auth caveat:** MCP auth rides the org's auth flow — orgs that do **not** sign in via Google/GitHub/GitLab may find the MCP IDE workflows don't work; confirm IdP compatibility before rollout. Capabilities mirror Cursor (replace base images, translate OS packages to Wolfi equivalents, update package-manager config, explain tradeoffs).

### Migrate a Dockerfile end-to-end (8-step checklist)
Manual fallback when the Guardener is unavailable or overkill:
```bash
# 1. Check the per-image overview at images.chainguard.dev/?image=<name>
# 2. Swap base image to the -dev variant
sed -i.bak 's|^FROM .*python.*|FROM cgr.dev/chainguard/python:latest-dev|' Dockerfile
# 3. Add USER root before package operations
# 4. Replace apt/yum install with apk add
# 5. apk search cmd:<binary>, apk -R info <pkg>, apk search so:<lib>.so* to find replacements
# 6. Set correct file permissions when copying app files (chown to non-root user)
# 7. Switch back to a non-root user before finalizing
# 8. Build, test, then optionally split into multi-stage with distroless runtime

# Quick first-pass alternative (offline, rule-based):
dfc --org $ORG Dockerfile
```

---

## Terraform — `chainguard-dev/chainguard` provider

For users managing IAM in Terraform rather than `chainctl` directly. Provider source: `chainguard-dev/chainguard`. The provider auto-launches browser OAuth if no token is cached; steer it with a `login_options` block.

```hcl
provider "chainguard" {
  login_options {
    organization_name = "example.com"     # uses the org's custom IDP for login
    identity_id       = "<UIDP>"          # assume this identity
    # CI: disable interactive login and consume an injected OIDC token instead
    # disabled       = true
    # identity_token = "/var/run/oidc/token"
  }
}
```

**Resources:** `chainguard_identity`, `chainguard_rolebinding`, `chainguard_role`, `chainguard_identity_provider`, `chainguard_group`.
**Data sources:** `chainguard_group`, `chainguard_identity`, `chainguard_role`, `chainguard_roles`.

**Naming/idiom notes that trip people up:**
- "Groups" in the provider == organizations/folders in current IAM terminology.
- `chainguard_group` with `parent_id = "/"` queries **root organizations** specifically.
- Role data sources return a **list** — use `.items[0].id`, not a single object.
- Resource arg naming differs from CLI: `parent_id` (not `--parent`), `claim_match { issuer_pattern, subject_pattern, audience_pattern }` block (not separate `--issuer`/`--subject` flags).
- Pre-binding human users requires subject prefixes: GitHub `github|${github_id}` (issuer `https://auth.chainguard.dev/`), Google `google-oauth2|<id>`, GitLab similar.
- Enumerate capability strings for `chainguard_role.capabilities` with `chainctl iam roles capabilities list`.

**GitHub Team → Chainguard role-binding pattern** uses the GitHub provider's `github_team` data source + a `for_each = toset(data.github_team.team.members)` block, then `chainguard_identity` + `chainguard_rolebinding` per member. Env: `GITHUB_ORG`, `GITHUB_TEAM` (slug — find with `gh api /orgs/<org>/teams`), `GITHUB_TOKEN` (PAT, min `read:org`, SSO-enabled if required), `CHAINGUARD_ORG` (UIDP).

---

## chainctl vs Console — when to use which

The Console is best for one-off exploration: browsing the image catalog, viewing CVE Visualizations (compare a Chainguard image against a non-Chainguard one — **Console-only**), looking at activity center / notifications setup, drilling into Custom Assembly UI flows.

`chainctl` is what you reach for when you know what you want: scripting, GitOps, anything CI, and a few capabilities the Console doesn't surface (`chainctl images diff` is **chainctl-only**).

---

## Tips

1. **Output formats** — Use `-o json` for scripting, `-o table` for readability, `-o tree` for hierarchy, `-o wide` for all fields, `-o id` for just IDs.
2. **Aliases** — Many commands have short aliases: `orgs`, `img`, `pkg`, `libs`, etc. Singular and plural both work for most resources (`images`/`image`, `repos`/`repo`, `identities`/`identity`).
3. **Get help** — Append `--help` to any command for detailed flag info. The published `chainctl` reference lives at `https://edu.chainguard.dev/chainguard/chainctl/`. **Note**: `chainctl agent dockerfile *` and `chainctl completion *` are not in the published reference; they work but were verified via `--help`. The reference doesn't list "Required Capabilities" per command — the canonical source for those is `chainctl iam roles capabilities list` and the Console capabilities matrix.
4. **LLM-friendly docs index** — `https://edu.chainguard.dev/llms.txt` is the complete documentation index in a machine-readable form. Every Chainguard doc page also exposes a "Copy Markdown for LLMs" button.
5. **Config file** — Set `CHAINCTL_CONFIG` env var or use `--config` to point to a specific config.
6. **Headless/CI** — Use `--headless` for non-interactive login, `--identity-token` for CI/CD pipelines. Activation code TTL is 900 seconds.
7. **Pull tokens** — For CI/CD image pulling, create pull tokens with appropriate TTL and save to Docker config. Pulls via `--pull-token` are attributed to the **identity**, not the user — relevant for audit logs.
8. **Registry status page** — `https://status.cgr.dev` for live registry incidents. `cgr.dev` does **not** have an uptime SLA — production workloads should pull through a cache (ECR, GAR, Artifactory, Nexus, Cloudsmith).
9. **`setup-chainctl` GitHub Action** — for GitHub Actions workflows, the official action is `chainguard-dev/setup-chainctl@v0.2.4` (takes an `identity:` input).
