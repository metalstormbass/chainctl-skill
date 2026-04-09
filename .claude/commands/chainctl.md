---
description: "Chainguard chainctl CLI assistant — helps construct, explain, and troubleshoot chainctl commands for managing Chainguard container images, IAM, authentication, and platform configuration."
allowed-tools: [Bash, Read, Grep, Glob, WebFetch]
---

You are a chainctl expert assistant. When the user asks about chainctl, help them construct the correct command, explain flags, troubleshoot errors, or accomplish their goal on the Chainguard platform.

**Always use long timeouts for chainctl commands.** Many chainctl operations are slow. Use `timeout: 300000` (5 minutes) for most commands. For `chainctl agent dockerfile` (The Guardener) commands use `timeout: 600000` (10 minutes). For `chainctl libraries verify` use `timeout: 600000` (10 minutes). Never use the default 2-minute timeout for chainctl.

**Always check for updates first.** Before doing anything else, run `chainctl update` to ensure the latest version is installed. Do this once at the start of every conversation.

**Always verify chainctl is available** by running `which chainctl` before suggesting commands. If the user asks you to run a command, confirm destructive operations (delete, reset, delete-account) before executing.

**Always ask for the organization name** at the start of the conversation if the user hasn't provided one. Many chainctl commands require a `--parent` or `--group` flag. Ask once, remember it, and use it for all subsequent commands in the session.

**Custom Assembly: Always use the file-based workflow.** The interactive editor (`chainctl images repos build edit` without `--file`) opens a terminal editor that does not work in Claude Code. Instead:
1. **Ask the user what they want to name the YAML config file** before creating it (e.g., `node-custom.yaml`, `my-python-build.yaml`). Always ask — never assume a default name.
2. Write the YAML config to the file with the user's chosen name.
3. **Always create a new image — never modify the base image.** Apply with `--save-as` to create a new repo: `chainctl images repos build apply --repo=<base-image> --file=<filename>.yaml --parent <org> --save-as=<new-name> --yes` (always use `apply` with `--yes` to avoid interactive prompts). If the user wants to update an existing custom image, use `--repo=<custom-image>` without `--save-as`.

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
| `--log-level` | Log level: `debug`, `info` |
| `-o, --output` | Output format: `csv`, `env`, `go-template`, `id`, `json`, `markdown`, `none`, `table`, `terse`, `tree`, `wide` |
| `-v, --v` | Log verbosity level |

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

### `chainctl auth logout`
Logout from the Chainguard platform. No special flags.

### `chainctl auth status`
Inspect the local Chainguard token.

**Flags:**
- `--quick` — Perform quick offline token checks (vs. calling the Validate API)
- Same auth flags as login (`--headless`, `--identity`, `--identity-provider`, `--identity-token`, `--org-name`, `--social-login`)

### `chainctl auth configure-docker`
Configure a Docker credential helper for pulling Chainguard images.

**Flags:**
- `--pull-token` — Register a pull token that can pull images
- `--save` — With `--pull-token`, save the pull token to Docker config
- `--parent` — IAM org or folder for the pull-token identity
- `--name` — Optional name for the pull token
- `--ttl` — Time To Live for pull token validity (units: `ns`, `us`, `ms`, `s`, `m`, `h`; max `8760h`/1 year)
- Auth flags: `--headless`, `--identity`, `--identity-provider`, `--identity-token`, `--org-name`, `--social-login`

**Required Capabilities:** `groups.list`, `roles.list`, `role_bindings.create`, `identity.create`, `libraries.entitlements.list`

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
Print the local Chainguard token. Has subcommand `capabilities` to print token capabilities.

### `chainctl auth pull-token`
Manage pull tokens. Aliases: `pull-tokens`.

**Subcommands:** `create`, `list`

#### `chainctl auth pull-token create`
Create a pull token. Aliases: `make`, `mk`.

**Flags:**
- `--name` — Optional name for the pull token
- `--parent` — IAM org or folder for the pull token identity
- `--repository` — Repository type: `oci`, `apk`, `java`, `python`, `javascript`
- `--save` — Save the OCI registry pull token to Docker configuration
- `--ttl` — Time To Live (max `8760h`/1 year)

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
Edit the current chainctl config file in your editor.

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

### `chainctl config save`
Save the current chainctl config to a config file.

### `chainctl config validate`
Run diagnostics on local config.

---

## iam — Identity and Access Management

### Organizations
`chainctl iam organizations` (aliases: `orgs`, `org`)

| Command | Description |
|---------|-------------|
| `list` | List organizations |
| `describe` | Describe an organization |
| `delete` | Delete an organization |

### Folders
`chainctl iam folders` (aliases: `folder`)

| Command | Description |
|---------|-------------|
| `list` | List folders under an organization |
| `describe` | Describe a folder |
| `delete` | Delete a folder |
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
`chainctl iam roles` (aliases: `role`)

| Command | Description |
|---------|-------------|
| `list` | List IAM roles |
| `create` | Create an IAM role |
| `delete` | Delete a custom IAM role |
| `update` | Update an IAM role |
| `capabilities list` | List IAM role capabilities |

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
`chainctl iam role-bindings` (aliases: `role-binding`, `rolebindings`, `rolebinding`)

| Command | Description |
|---------|-------------|
| `list` | List role-bindings |
| `create` | Create a role-binding |
| `delete` | Delete a role-binding |
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
`chainctl iam invites` (aliases: `invite`)

| Command | Description |
|---------|-------------|
| `list` | List organization and folder invites |
| `create` | Generate an invite code to register identities |
| `delete` | Delete invite codes |

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
`chainctl iam identity-providers`

| Command | Description |
|---------|-------------|
| `list` | List identity providers |
| `create` | Create a customer managed identity provider |
| `delete` | Delete an identity provider |
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
`chainctl iam account-associations` (aliases: `accountassociations`)

Configure cloud provider account associations (AWS, Azure, GCP).

| Command | Description |
|---------|-------------|
| `describe` | Describe cloud provider account associations for a location |
| `check aws\|gcp\|azure` | Check OIDC federation configurations |
| `set aws\|gcp\|azure` | Set cloud provider account associations |
| `unset aws\|gcp\|azure` | Remove cloud provider account associations |

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
Diff two images based on SBOM and vulnerability scan. Requires `grype` on PATH.

**Flags:**
- `-t, --artifact-types` — PURL artifact types to diff (use `-` for all)
- `--platform` — Platform in `os/arch` format (e.g. `linux/amd64`)
- `--template` — Go template for `--output=go-template`
- `--template-file` — Path to Go template file

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

- `list` — List tags from repositories (use `--parent`, `--public`, or `--repo`)
- `resolve` — Resolve tags for a specific image reference

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
- `--tier` — Catalog tier: `BASE`, `FIPS`, `AI`, `DEVTOOLS`, `APPLICATION`
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
| `annotations` | Custom OCI annotations (`dev.chainguard` prefix is reserved) |
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
- `-f, --file` — Config file to apply (required)
- `--save-as` — Create a new repo instead of updating existing
- `--with-certificates` — Certificate files to include
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

### `chainctl images entitlements list`
List registry entitlements of an organization.

### `chainctl images helm values`
Generate relocation overrides for a Chainguard Helm chart.

---

## events — Event Subscriptions

`chainctl events` (aliases: `event`)

### `chainctl events subscriptions`
Aliases: `subscription`, `subs`, `sub`.

| Command | Description |
|---------|-------------|
| `list` | List subscriptions |
| `create` | Subscribe to events under an organization or folder |
| `delete` | Delete a subscription |

#### `chainctl events subscriptions create`
Subscribe to events. Aliases: `make`, `mk`.

**Flags:**
- `--parent` — Parent location name or ID for the subscription
- `-y, --yes` — Auto-confirm prompts

**Required Capabilities:** `groups.list`, `subscriptions.create`

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

**Supports:** directories, archives, packages, container images (registry refs, local images, docker-archive format).

**Flags:**
- `-d, --detailed` — Show detailed per-artifact results
- `--no-color` — Disable colored output
- `-o, --output` — Output format: `text`, `json`, `yaml`
- `--verbose` — Enable verbose output

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
- `--ecosystems` — Language ecosystems to entitle: `JAVASCRIPT`, `JAVA`, `PYTHON` (comma-separated for multiple)
- `--parent` — Name or ID of the org to create an entitlement for
- `--policy` — Policy to apply: `CHAINGUARD` (Chainguard-only) or `CHAINGUARD_AND_UPSTREAM` (Chainguard repo with upstream fallback; **currently only supported for JAVASCRIPT**)

**Required Capabilities:** `groups.list`, `libraries.entitlements.create`

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

- `chainctl update` — Update chainctl to the latest version
- `chainctl version` — Print the current version

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
```

---

## Tips

1. **Output formats** — Use `-o json` for scripting, `-o table` for readability, `-o tree` for hierarchy, `-o wide` for all fields, `-o id` for just IDs.
2. **Aliases** — Many commands have short aliases: `orgs`, `img`, `pkg`, `libs`, etc.
3. **Get help** — Append `--help` to any command for detailed flag info.
4. **Config file** — Set `CHAINCTL_CONFIG` env var or use `--config` to point to a specific config.
5. **Headless/CI** — Use `--headless` for non-interactive login, `--identity-token` for CI/CD pipelines.
6. **Pull tokens** — For CI/CD image pulling, create pull tokens with appropriate TTL and save to Docker config.
