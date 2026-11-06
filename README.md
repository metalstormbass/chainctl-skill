# chainctl Skill for Claude Code

A custom [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash command that turns Claude into a **chainctl expert assistant** — helping you construct, explain, troubleshoot, and execute [`chainctl`](https://www.chainguard.dev/docs/platform/chainctl) commands for the [Chainguard](https://www.chainguard.dev/) platform.

## What is chainctl?

`chainctl` (Chainguard Control) is the official CLI for the Chainguard platform. It provides command-line access to:

- **Authentication** — Login, logout, session management (`auth sessions`), token management, Docker/npm/NuGet credential helpers, pull tokens
- **Container Images** — List, diff, history, changelog, tags, advisories, repos, Helm chart values
- **Custom Assembly** — Customize Chainguard images by adding packages, env vars, OCI annotations, user accounts, and certificates without maintaining custom build pipelines; tag-based overlays (`chainctl images overlays`, chainctl ≥ 0.2.389, selector flags `--all-tags`/`--tag-variant`/`--exact-tag`) and Guarded Entrypoint (beta: secrets, preflight checks, command override, runtime Java truststore)
- **IAM** — Organizations, folders, identities, roles, role-bindings, invites, identity providers (incl. SCIM group push for Okta/Entra), org session-length auth policy, cloud account associations (AWS, Azure, GCP)
- **Events** — Subscription management for platform events
- **Packages** — Query package version data from Chainguard repositories
- **Libraries** — Verify artifacts are built from Chainguard sources using SBOM/signature analysis; update lockfile hashes, remediate, manage library policies (cooldown, overrides), entitlements, the resolution cache, and the malware blocklist
- **Policies** — Registry pull-time policy governance: enable/disable policies, custom policies, bindings, decisions, overrides
- **Guardener** — Chainguard App (GitHub) integration: link GitHub orgs to Chainguard groups, migrate GitHub Actions workflows and container image references to Chainguard equivalents
- **Skills** — Publish, install, and manage agent skills on the Skills Registry (`skills.cgr.dev`)
- **Actions & Starter** — Chainguard Actions catalog/entitlements and Catalog Starter org setup
- **Checks** — Chainguard Checks (beta): CI that runs each check in a fresh Chainguard OS microVM. Covers the `.chainguard/ci.yaml` / `.chainguard/ci/` config, GitHub integration, workload identities (GCP, AWS, Azure), caches (Go, Python, Java, keyed), the 2-hour run limit, and `chainctl checks` (alias `ci`) for status/failures/logs/rerun/validate/explain, plus `checks mcp` for MCP clients
- **Workspaces** — Chainguard Workspaces (beta): remote microVM dev sessions for repos and coding agents via `chainctl develop`. Covers sign-up with `chainctl develop init`, sessions, environments, brokered credentials, GitHub-as-you, Claude Code, `develop exec`, `develop mcp`, and `run-ci`
- **Plans** — Org plan, limits, and quota usage (`chainctl plans`)
- **Compliance & supply chain** — FIPS verification, STIG, signature/attestation verification (incl. the Sigstore v3 signing transition), Kyverno/Gatekeeper enforcement recipes
- **MCP servers** — cg-api, cg-oci, cg-apk, cg-versions, and the public cg-man, AI Docs, and Public Skills servers
- **Configuration** — View, edit, set, validate local chainctl config

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed
- [`chainctl`](https://www.chainguard.dev/docs/platform/chainctl-usage/how-to-install-chainctl) installed and on your `PATH`

**Install chainctl (macOS):**
```bash
brew install chainguard-dev/tap/chainctl
```

**Install chainctl (Linux):**
```bash
PLATFORM=$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/aarch64/arm64/')
VERSION=$(curl -sS https://dl.enforce.dev/chainctl/latest/metadata.json | jq -r .version)
curl -fL -o chainctl "https://dl.enforce.dev/chainctl/${VERSION}/chainctl_${PLATFORM}"
sudo install -o "$UID" -g "$(id -g)" -m 0755 chainctl /usr/local/bin/
```

### Option A: Clone and use directly

Clone this repo and run Claude Code from inside it:

```bash
git clone https://github.com/metalstormbass/chainctl-skill.git
cd chainctl-skill
claude
```

The `/chainctl` slash command is immediately available. The included `settings.local.json` pre-approves chainctl commands so Claude can run them without repeated prompts.

### Option B: Copy into an existing project

Copy the `.claude` directory into any project where you want the skill available:

```bash
git clone https://github.com/metalstormbass/chainctl-skill.git
cp -r chainctl-skill/.claude/commands/ /path/to/your/project/.claude/commands/
```

Optionally copy the permissions config too:

```bash
cp chainctl-skill/.claude/settings.local.json /path/to/your/project/.claude/settings.local.json
```

> **Note:** If your project already has a `settings.local.json`, merge the `permissions.allow` entries manually instead of overwriting.

### Option C: Install globally

Make the skill available across all your projects:

```bash
git clone https://github.com/metalstormbass/chainctl-skill.git
mkdir -p ~/.claude/commands
cp chainctl-skill/.claude/commands/chainctl.md ~/.claude/commands/chainctl.md
```

> **Note:** Global installation does not include the permissions config. You will be prompted to approve chainctl commands individually.

## Usage

Once installed, invoke the skill inside Claude Code:

```
/chainctl
```

Then ask Claude anything about chainctl:

- *"How do I set up Docker to pull Chainguard images in CI?"*
- *"List all images in my org as JSON"*
- *"Create a custom assembly config that adds curl and jq to the python image"*
- *"What capabilities does my token have?"*
- *"Set up a GitHub Actions identity for my org"*
- *"Show me the changelog for the nginx image"*
- *"How do I configure cloud account associations for AWS?"*

Claude will construct the correct command with all appropriate flags, explain what it does, and optionally execute it for you.

## Examples

### Quick command help
```
> /chainctl
> How do I login with a GitHub identity?

chainctl auth login --social-login github
```

### CI/CD setup
```
> /chainctl
> Set up Docker auth for a CI pipeline with a 30-day pull token

chainctl auth configure-docker --pull-token --save --parent my-org --ttl 720h
```

### Custom Assembly
```
> /chainctl
> Customize the python image to add curl and git

chainctl images repos build edit --repo=python --save-as=my-custom-python --parent my-org
# Opens editor with YAML config where you add packages under contents.packages
```

### Image investigation
```
> /chainctl
> What changed in the last 5 versions of the nginx image?

chainctl images changelog cgr.dev/chainguard/nginx:latest --depth 5
```

### IAM exploration
```
> /chainctl
> Show me all identities and role-bindings in my org as JSON

chainctl iam identities list --parent my-org -o json
chainctl iam role-bindings list --parent my-org -o json
```

## What the Skill Covers

### Command Groups

| Command | Description | Subcommands |
|---------|-------------|-------------|
| `chainctl auth` | Authentication & token management | `login`, `logout`, `status`, `configure-docker`, `configure-npm`, `configure` (`docker`, `npm`, `nuget`), `configure-refresh-service`, `token` (+ `capabilities`, delegation grants), `pull-token`, `sessions` (`list`, `revoke`), `delete-account` |
| `chainctl config` | Local configuration | `view`, `edit`, `set`, `unset`, `reset`, `save`, `validate` |
| `chainctl iam` | Identity & access management | `organizations`, `folders`, `identities`, `roles`, `role-bindings`, `invites`, `identity-providers`, `external-group-role-mappings`, `account-associations` (`set`, `unset`, `check`, `describe`), `auth-policy` (`describe`, `update`) |
| `chainctl images` | Container image operations | `list`, `diff`, `history`, `changelog`, `tags`, `repos`, `overlays`, `advisories`, `entitlements` (`list`, `summary`, `add-image`, `remove-image`, `swap-image`), `catalog`, `discover`, `helm` (`values`, `refs`, `add-chart`, `history`) |
| `chainctl events` | Event subscriptions | `subscriptions` (`list`, `create`, `delete`) |
| `chainctl packages` | Package management | `versions list` |
| `chainctl libraries` | Libraries verification & governance | `verify`, `update-hashes`, `remediate`, `packages` (`list`, `count`, `versions`, `blocked`, `malware list`), `policy` (+ `binding`), `entitlements`, `cache` (`list`, `status`, `opt-in`, `opt-out`, `zap`), `go upload`, `storage list` |
| `chainctl policy` (alias `policies`) | Registry pull-time policy governance | `list`, `describe`, `enable`, `disable`, `check`, `binding`, `decision`, `override`, `custom` |
| `chainctl actions` | Chainguard Actions product | `entitlements`, `catalog list`, `list`, `discover` |
| `chainctl agent` | The Guardener (local Dockerfile migration) | `accept-terms`, `dockerfile` (`build`, `optimize`, `upgrade`, `validate` — hidden from `chainctl agent --help` but functional) |
| `chainctl guardener` | Guardener GitHub App integration | `github` (`link`, `unlink`, `status`, `migrate` (`create`, `get`, `list`)), `scans` (`list`, `get`), `entitlement get` |
| `chainctl skills` | Skills Registry (`skills.cgr.dev`) | `push`, `pull`, `install`, `uninstall`, `list`, `versions`, `describe`, `validate`, `verify`, `harden`, `status`, `registry` (`publish`, `install`), `delete`, `entitlements`, `accept-terms` |
| `chainctl starter` | Catalog Starter orgs | `init`, `accept-terms`, `request-access`, `add-images`, `status` |
| `chainctl develop` (alias `dev`) | Chainguard Workspaces dev sessions (beta; listed in root help since v0.2.391) | `init`, `accept-terms`, `list`, `rename`, `delete`, `exec`, `check`, `creds`, `policy` (`describe`, `set`), `github` (`login`, `logout`, `status`), `claude` (`login`, `logout`, `status`), `mcp` |
| `chainctl checks` (alias `ci`) | Chainguard Checks (beta; listed in root help since v0.2.391) | `status`, `failures`, `logs`, `trace`, `events`, `attempts`, `describe`, `list`, `run`, `rerun`, `cancel`, `stats`, `top`, `trends`, `validate`, `explain`, `mcp`, `accept-terms`, `migrate` (hidden) |
| `chainctl plans` | Org plan, limits & usage | `describe`, `quotas list` |
| `chainctl update` (alias `upgrade`) | Self-update | — (`--yes`, `--force`) |
| `chainctl version` | Print version | — |

### Custom Assembly

The skill includes detailed documentation for Chainguard's Custom Assembly feature:

- **`chainctl images repos build edit`** — Interactive YAML editor for image customization
- **`chainctl images repos build apply`** — Non-interactive config application (CI/CD)
- **`chainctl images repos build list`** — List build reports
- **`chainctl images repos build logs`** — Retrieve build logs

### Additional Features

- All global flags documented (`--output`, `--api`, `--config`, etc.)
- 11 output formats explained with usage guidance
- Command aliases (e.g., `orgs`, `img`, `pkg`, `libs`, `rm`, `ls`, `mk`)
- Required IAM capabilities listed per command
- Workflow recipes: first-time setup, Docker config, image exploration, IAM management, Custom Assembly, library verification, choosing how to add a package, sizing a migration (`images discover` → Image Matcher → dfc/Guardener), and an "image/version not pullable" decision tree
- Exit-code reference for scripting (root `chainctl --help`)
- Verified against chainctl v0.2.397 and the www.chainguard.dev/docs tree (2026-11-06)
- Safety guards: confirms destructive operations before execution
- Verifies `chainctl` availability before suggesting commands

## Permissions Configuration

The included `settings.local.json` pre-approves these tool calls so Claude can run chainctl commands without repeated prompts:

```json
{
  "permissions": {
    "allow": [
      "WebFetch(domain:edu.chainguard.dev)",
      "Bash(chainctl auth:*)",
      "Bash(chainctl iam:*)",
      "Bash(chainctl images:*)",
      "Bash(chainctl config:*)",
      "Bash(chainctl events:*)",
      "Bash(chainctl packages:*)",
      "Bash(chainctl libraries:*)",
      "WebSearch"
    ]
  }
}
```

This allows Claude to:
- Run any `chainctl` subcommand
- Fetch documentation from `edu.chainguard.dev` (that host now redirects to `www.chainguard.dev/docs`; add `"WebFetch(domain:www.chainguard.dev)"` to the allowlist to fetch the new location without prompts)
- Search the web for additional Chainguard context

> **Note:** Destructive commands (`delete`, `reset`, `delete-account`) are allowed at the CLI permission level but the skill prompt instructs Claude to **always confirm** before executing them.

## Project Structure

```
chainctl-skill/
├── .gitignore
├── README.md
└── .claude/
    ├── settings.local.json            # Permission allowlists for chainctl commands
    └── commands/
        └── chainctl.md               # The skill definition
```

## Extending the Skill

To add coverage for new chainctl commands or features:

1. Edit `.claude/commands/chainctl.md`
2. Add the command under the appropriate section following the existing format:
   - Command name and description
   - Flags with descriptions
   - Required capabilities (if any)
   - Examples
3. If it represents a common workflow, add a recipe to the **Common Workflows** section

To update from the latest chainctl help:
```bash
chainctl <new-command> --help
```

## Resources

- [Chainguard Docs — chainctl Usage](https://www.chainguard.dev/docs/platform/chainctl-usage)
- [Chainguard Docs — chainctl Reference](https://www.chainguard.dev/docs/platform/chainctl)
- [Custom Assembly Documentation](https://www.chainguard.dev/docs/chainguard/containers/custom-assembly/overview)
- [Chainguard products changelog](https://www.chainguard.dev/docs/chainguard/changelog)
- [Docs index for LLMs (`llms.txt`)](https://www.chainguard.dev/docs/llms.txt)
- [chainctl Roles & Capabilities](https://www.chainguard.dev/docs/platform/administration/iam-organizations/roles-role-bindings/capabilities-reference)
- [Chainguard Platform Console](https://console.chainguard.dev)

## License

MIT
