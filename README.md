# chainctl Skill for Claude Code

A custom [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash command that turns Claude into a **chainctl expert assistant** — helping you construct, explain, troubleshoot, and execute [`chainctl`](https://edu.chainguard.dev/chainguard/chainctl/) commands for the [Chainguard](https://www.chainguard.dev/) platform.

## What is chainctl?

`chainctl` (Chainguard Control) is the official CLI for the Chainguard platform. It provides command-line access to:

- **Authentication** — Login, logout, token management, Docker credential helpers, pull tokens
- **Container Images** — List, diff, history, changelog, tags, advisories, repos, Helm chart values
- **Custom Assembly** — Customize Chainguard images by adding packages, env vars, OCI annotations, user accounts, and certificates without maintaining custom build pipelines
- **IAM** — Organizations, folders, identities, roles, role-bindings, invites, identity providers, cloud account associations (AWS, Azure, GCP)
- **Events** — Subscription management for platform events
- **Packages** — Query package version data from Chainguard repositories
- **Libraries** — Verify artifacts are built from Chainguard sources using SBOM/signature analysis
- **Configuration** — View, edit, set, validate local chainctl config

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

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed
- [`chainctl`](https://edu.chainguard.dev/chainguard/chainctl-usage/how-to-install-chainctl/) installed and on your `PATH`

**Install chainctl (macOS):**
```bash
brew install chainguard-dev/tap/chainctl
```

**Install chainctl (Linux):**
```bash
wget -O chainctl "https://dl.enforce.dev/chainctl/latest/chainctl_linux_$(uname -m)"
chmod +x chainctl
sudo mv chainctl /usr/local/bin/
```

### Setup

1. Clone or copy this project:
   ```bash
   git clone <repo-url> chainctl_skill
   cd chainctl_skill
   ```

2. Open Claude Code from this directory:
   ```bash
   claude
   ```

3. The `/chainctl` slash command is now available. The skill is loaded from `.claude/commands/chainctl.md`.

### Alternative: Install globally

To make the skill available in all your projects, copy the command file to your global Claude Code config:

```bash
mkdir -p ~/.claude/commands
cp .claude/commands/chainctl.md ~/.claude/commands/chainctl.md
```

## Project Structure

```
chainctl_skill/
├── README.md                          # This file
└── .claude/
    ├── settings.local.json            # Permission allowlists for chainctl commands
    └── commands/
        └── chainctl.md               # The skill definition (694 lines)
```

### File Details

| File | Purpose |
|------|---------|
| `.claude/commands/chainctl.md` | The skill prompt — comprehensive chainctl reference with every command, flag, example, and workflow recipe. Loaded when you run `/chainctl`. |
| `.claude/settings.local.json` | Pre-configured permissions allowing Claude to run `chainctl` subcommands and fetch Chainguard documentation without repeated approval prompts. |

## What the Skill Covers

### Command Groups (7 top-level + utilities)

| Command | Description | Subcommands |
|---------|-------------|-------------|
| `chainctl auth` | Authentication & token management | `login`, `logout`, `status`, `configure-docker`, `token`, `pull-token`, `delete-account` |
| `chainctl config` | Local configuration | `view`, `edit`, `set`, `unset`, `reset`, `save`, `validate` |
| `chainctl iam` | Identity & access management | `organizations`, `folders`, `identities`, `roles`, `role-bindings`, `invites`, `identity-providers`, `account-associations` |
| `chainctl images` | Container image operations | `list`, `diff`, `history`, `changelog`, `tags`, `repos`, `advisories`, `entitlements`, `helm` |
| `chainctl events` | Event subscriptions | `subscriptions` (`list`, `create`, `delete`) |
| `chainctl packages` | Package management | `versions list` |
| `chainctl libraries` | Library verification | `verify` |
| `chainctl update` | Self-update | — |
| `chainctl version` | Print version | — |

### Custom Assembly (Deep Coverage)

The skill includes detailed documentation for Chainguard's Custom Assembly feature:

- **`chainctl images repos build edit`** — Interactive YAML editor for image customization
- **`chainctl images repos build apply`** — Non-interactive config application (CI/CD)
- **`chainctl images repos build list`** — List build reports
- **`chainctl images repos build logs`** — Retrieve build logs

**Customizable YAML sections:**

| Section | What You Can Configure |
|---------|----------------------|
| `contents.packages` | Additional packages from Chainguard's repo |
| `environment` | Environment variables (except `CHAINGUARD_*` prefix) |
| `annotations` | OCI annotations (except `dev.chainguard` prefix) |
| `accounts` | Custom users/groups, UIDs/GIDs, run-as user |
| `certificates` | Custom CA certs merged with default bundle (Beta) |

### Additional Skill Features

- **All global flags** documented (`--output`, `--api`, `--config`, etc.)
- **12 output formats** explained with usage guidance
- **Command aliases** (e.g., `orgs`, `img`, `pkg`, `libs`, `rm`, `ls`, `mk`)
- **Required IAM capabilities** listed per command
- **6 workflow recipes**: first-time setup, Docker config, image exploration, IAM management, Custom Assembly, library verification
- **Safety guards**: confirms destructive operations before execution
- **Verifies `chainctl` availability** before suggesting commands

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
- Fetch documentation from `edu.chainguard.dev`
- Search the web for additional Chainguard context

> **Note:** Destructive commands (`delete`, `reset`, `delete-account`) are allowed at the CLI permission level but the skill prompt instructs Claude to **always confirm** before executing them.

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

- [Chainguard Academy — chainctl Usage](https://edu.chainguard.dev/chainguard/chainctl-usage/)
- [Chainguard Academy — chainctl Reference](https://edu.chainguard.dev/chainguard/chainctl/)
- [Custom Assembly Documentation](https://edu.chainguard.dev/chainguard/chainguard-images/features/ca-docs/custom-assembly-chainctl/)
- [chainctl Roles & Capabilities](https://go.chainguard.dev/chainctl-roles)
- [Chainguard Platform Console](https://console.chainguard.dev)

## License

MIT
# chainctl-skill
