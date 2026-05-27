# Claude Code and Codex in a devcontainer

A sandboxed development environment for running Claude Code and OpenAI Codex CLI with their low-friction agent modes enabled. Built at [Trail of Bits](https://www.trailofbits.com/) for security audit workflows.

## Why Use This?

Running coding agents with unrestricted command execution on your host machine is risky. Claude Code `bypassPermissions` and Codex `danger-full-access` can execute commands, install packages, and modify files without stopping for confirmation. This devcontainer provides **filesystem isolation** so the agent can work freely inside `/workspace` without getting broad access to your host.

**Designed for:**

- **Security audits**: Review client code without risking your host
- **Untrusted repositories**: Explore unknown codebases safely
- **Experimental work**: Let agents modify code freely in isolation
- **Multi-repo engagements**: Work on multiple related repositories

## Prerequisites

- **Docker runtime** (one of):
  - [Docker Desktop](https://docker.com/products/docker-desktop) - ensure it's running
  - [OrbStack](https://orbstack.dev/)
  - [Colima](https://github.com/abiosoft/colima): `brew install colima docker && colima start`

- **For terminal workflows** (one-time install):

  ```bash
  npm install -g @devcontainers/cli
  git clone https://github.com/trailofbits/claude-code-devcontainer ~/.claude-devcontainer
  ~/.claude-devcontainer/install.sh self-install
  ```

<details>
<summary><strong>Optimizing Colima for Apple Silicon</strong></summary>

Colima's defaults (QEMU + sshfs) are conservative. For better performance:

```bash
# Stop and delete current VM (removes containers/images)
colima stop && colima delete

# Start with optimized settings
colima start \
  --cpu 4 \
  --memory 8 \
  --disk 100 \
  --vm-type vz \
  --vz-rosetta \
  --mount-type virtiofs
```

Adjust `--cpu` and `--memory` based on your Mac (e.g., 6/16 for Pro, 8/32 for Max).

| Option | Benefit |
|--------|---------|
| `--vm-type vz` | Apple Virtualization.framework (faster than QEMU) |
| `--mount-type virtiofs` | 5-10x faster file I/O than sshfs |
| `--vz-rosetta` | Run x86 containers via Rosetta |

Verify with `colima status` - should show "macOS Virtualization.Framework" and "virtiofs".

</details>

## Quick Start

Choose the pattern that fits your workflow:

### Pattern A: Per-Project Container (Isolated)

Each project gets its own container with independent volumes. Best for one-off reviews, untrusted repos, or when you need isolation between projects.

**Terminal:**

```bash
git clone <untrusted-repo>
cd untrusted-repo
devc .          # Installs template + starts container
devc shell      # Opens shell in container

# Inside container:
claude          # Claude Code
codex           # OpenAI Codex CLI
```

**VS Code / Cursor:**

1. Install the Dev Containers extension:
   - VS Code: `ms-vscode-remote.remote-containers`
   - Cursor: `anysphere.remote-containers`

2. Set up the devcontainer (choose one):

   ```bash
   # Option A: Use devc (recommended)
   devc .

   # Option B: Clone manually
   git clone https://github.com/trailofbits/claude-code-devcontainer .devcontainer/
   ```

3. Open **your project folder** in VS Code, then:
   - Press `Cmd+Shift+P` (Mac) or `Ctrl+Shift+P` (Windows/Linux)
   - Type "Reopen in Container" and select **Dev Containers: Reopen in Container**

### Pattern B: Shared Workspace Container (Grouped)

A parent directory contains the devcontainer config, and you clone multiple repos inside. Shared volumes across all repos. Best for client engagements, related repositories, or ongoing work.

```bash
# Create workspace for a client engagement
mkdir -p ~/sandbox/client-name
cd ~/sandbox/client-name
devc .          # Install template + start container
devc shell      # Opens shell in container

# Inside container:
git clone <client-repo-1>
git clone <client-repo-2>
cd client-repo-1
codex           # or: claude
```

## Authentication

### Claude Code

Interactive Claude login works normally inside the container.

For headless servers or to skip the interactive login wizard:

```bash
claude setup-token                          # run on host, one-time
export CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat01-...
devc rebuild                                # rebuilds with token
```

The token is forwarded into the container. On each container creation, `post_install.py` runs a one-shot auth handshake so `claude` starts without the login wizard.

This works around Claude Code's interactive onboarding wizard always showing in containers, even with valid credentials ([#8938](https://github.com/anthropics/claude-code/issues/8938)).

### Codex

Interactive Codex login works normally inside the container:

```bash
codex
```

Codex will prompt you to sign in with ChatGPT or an API key. Its auth and config are stored in the persistent `~/.codex` volume.

The container also adds the Trail of Bits Codex plugin marketplace with `codex plugin marketplace add trailofbits/skills`.

For headless use, export one of these on the host before creating or rebuilding the container:

```bash
export CODEX_ACCESS_TOKEN=<codex-access-token>
# or
export OPENAI_API_KEY=<openai-api-key>
devc rebuild
```

If both are set, `CODEX_ACCESS_TOKEN` wins. `post_install.py` persists the credential with `codex login --with-access-token` or `codex login --with-api-key`.

## CLI Helper Commands

```
devc .                         Install template + start container in current directory
devc up                        Start the devcontainer
devc rebuild                   Rebuild container (preserves persistent volumes)
devc destroy [-f]              Remove container, volumes, and image for current project
devc down                      Stop the container
devc shell                     Open zsh shell in container
devc exec CMD                  Execute command inside the container
devc upgrade [claude|codex|all] Upgrade agent CLI(s), defaults to all
devc mount SRC DST             Add a bind mount (host -> container)
devc sync [NAME]               Sync Claude Code sessions from devcontainers to host
devc template DIR              Copy devcontainer files to directory
devc self-install              Install devc to ~/.local/bin
```

> **Note:** Use `devc destroy` to clean up a project's Docker resources. Removing containers manually (e.g., `docker rm`) will leave orphaned volumes and images behind that `devc destroy` won't be able to find.

## Claude Session Sync for `/insights`

Claude Code's `/insights` command analyzes your session history, but it only reads from `~/.claude/projects/` on the host. Sessions inside devcontainer volumes are invisible to it.

`devc sync` copies Claude session logs from all devcontainers (running and stopped) to the host so `/insights` can include them:

```bash
devc sync              # Sync all devcontainers
devc sync crypto       # Filter by project name (substring match)
```

Devcontainers are auto-discovered via Docker labels. The sync is incremental, so it's safe to run repeatedly.

## File Sharing

### VS Code / Cursor

Drag files from your host into the VS Code Explorer panel. They are copied into `/workspace/` automatically.

### Terminal: `devc mount`

To make a host directory available inside the container:

```bash
devc mount ~/drop /drop           # Read-write
devc mount ~/secrets /secrets --readonly
```

This adds a bind mount to `devcontainer.json` and recreates the container. Existing mounts are preserved across `devc template` updates.

**Tip:** A shared "drop folder" is useful for passing files in without mounting your entire home directory.

> **Security note:** Avoid mounting large host directories (e.g., `$HOME`). Every mounted path is writable from inside the container unless `--readonly` is specified, which undermines the filesystem isolation this project provides.

## Network Isolation

By default, containers have full outbound network access. For stricter security, use iptables to restrict network access.

### When to Enable Network Isolation

- Reviewing code that may contain malicious dependencies
- Auditing software with telemetry or phone-home behavior
- Maximum isolation for highly sensitive reviews

### Example: Agents + GitHub + Package Registries

```bash
sudo iptables -A OUTPUT -d api.anthropic.com -j ACCEPT
sudo iptables -A OUTPUT -d api.openai.com -j ACCEPT
sudo iptables -A OUTPUT -d chatgpt.com -j ACCEPT
sudo iptables -A OUTPUT -d auth.openai.com -j ACCEPT
sudo iptables -A OUTPUT -d github.com -j ACCEPT
sudo iptables -A OUTPUT -d raw.githubusercontent.com -j ACCEPT
sudo iptables -A OUTPUT -d registry.npmjs.org -j ACCEPT
sudo iptables -A OUTPUT -d pypi.org -j ACCEPT
sudo iptables -A OUTPUT -d files.pythonhosted.org -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT
sudo iptables -A OUTPUT -j DROP
```

### Trade-offs

- Blocks package managers unless you allowlist registries
- May break tools that require network access
- DNS resolution still works (consider blocking if paranoid)

## Threat Model

The primary threat this project addresses is **coding agents running arbitrary commands on your host machine**. On a host machine, unrestricted agent modes can modify your shell config, delete files outside the project directory, or abuse locally stored credentials. The devcontainer confines that activity to a disposable container where the blast radius is limited to `/workspace` and any explicit mounts you add.

The container auto-configures:

- Claude Code `permissions.defaultMode = bypassPermissions`
- Codex `approval_policy = "never"` and `sandbox_mode = "danger-full-access"`

This would be risky on a host machine, but the container itself is the sandbox.

The container includes common development tooling so you can do all development work inside it - not just run agents. The intended workflow is: clone a repository, start the devcontainer, and work entirely within it. If your project needs additional runtimes or tools beyond what's included, either add them to the Dockerfile for repeated use or install them ad-hoc with `devc exec`.

For the specific boundaries of what is and isn't isolated, see [Security Model](#security-model) below. One nuance worth calling out: the devcontainer runtime automatically forwards your host's SSH agent socket (`SSH_AUTH_SOCK`) into the container. This lets code inside the container authenticate as you over SSH (e.g., `git push`), but the actual private key material stays on the host and is never exposed to the container.

## Security Model

This devcontainer provides **filesystem isolation** but not complete sandboxing.

**Sandboxed:** Filesystem (host files inaccessible except explicit mounts), processes (isolated from host), package installations (stay in container)

**Not sandboxed:** Network (full outbound by default - see [Network Isolation](#network-isolation)), git identity (`~/.gitconfig` mounted read-only), SSH agent (socket forwarded, keys stay on host), Docker socket (not mounted by default)

## Container Details

| Component | Details |
|-----------|---------|
| Base | Ubuntu 24.04, Node.js 22, Python 3.13 + uv, zsh |
| User | `vscode` (passwordless sudo), working dir `/workspace` |
| Agent CLIs | Claude Code, OpenAI Codex CLI |
| Tools | `rg`, `fd`, `tmux`, `fzf`, `delta`, `iptables`, `ipset` |
| Volumes (survive rebuilds) | Command history (`/commandhistory`), Claude config (`~/.claude`), Codex config/auth (`~/.codex`), GitHub CLI auth (`~/.config/gh`) |
| Host mounts | `~/.gitconfig` (read-only), `.devcontainer/` (read-only) |
| Auto-configured | Claude skills, Codex Trail of Bits marketplace, Claude `bypassPermissions`, Codex no-approval `danger-full-access`, git-delta |

Volumes are stored outside the container, so your shell history, agent settings, and `gh` login persist even after `devc rebuild`. Host `~/.gitconfig` is mounted read-only for git identity.

## Troubleshooting

### "devcontainer CLI not found"

```bash
npm install -g @devcontainers/cli
```

### Container won't start

1. Check Docker is running
2. Try rebuilding: `devc rebuild`
3. Check logs: `docker logs $(docker ps -lq)`

### Codex auth not working

Check the login status inside the container:

```bash
codex login status
```

For a headless container, confirm `CODEX_ACCESS_TOKEN` or `OPENAI_API_KEY` is set on the host, then run `devc rebuild`.

### GitHub CLI auth not persisting

The gh volume may need ownership fix:

```bash
sudo chown -R $(id -u):$(id -g) ~/.config/gh
```

### Python/uv not working

Python is managed via uv:

```bash
uv run script.py              # Run a script
uv add package                # Add project dependency
uv run --with requests py.py  # Ad-hoc dependency
```

## Development

Build the image manually:

```bash
devcontainer build --workspace-folder .
```

Test the container:

```bash
devcontainer up --workspace-folder .
devcontainer exec --workspace-folder . zsh
```
