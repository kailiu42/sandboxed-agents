# sandboxed-agents

Unified [bwrap](https://github.com/containers/bubblewrap) sandbox wrapper for
multiple code agents — OMP, Claude Code, Codex, and anything else you add.

One script, per-agent profiles. No copy-paste.

## Usage

```bash
git clone <this-repo>
cd sandboxed-agents

# Run via symlink (agent auto-detected from binary name)
./omp               # starts OMP
./claude            # starts Claude Code
./codex             # starts Codex

# Run via --agent flag
./agent-wrapper --agent omp
./agent-wrapper --agent=codex

# Override binary path via environment variable
AGENT_BIN=/usr/bin/omp ./claude
```

## How it works

```
.
├── agent-wrapper              # sandbox engine (no need to edit)
├── config/
│   ├── common.conf            # shared sandbox policy
│   ├── omp.conf               # OMP profile
│   ├── claude.conf            # Claude Code profile
│   └── codex.conf             # Codex profile
├── omp -> agent-wrapper       # symlink
├── claude -> agent-wrapper    # symlink
└── codex -> agent-wrapper     # symlink
```

The wrapper determines the agent name from `$0` (symlink name) or `--agent`.
Configs are loaded in order; later sources override earlier ones:

1. `config/common.conf`           — project-level shared policy
2. `config/$AGENT.conf`           — project-level agent profile
3. `~/.config/sandboxed-agents/common.conf`  — user-level shared overrides
4. `~/.config/sandboxed-agents/$AGENT.conf`  — user-level agent overrides

## Override defaults

Place a config file in `~/.config/sandboxed-agents/` to add or override binds:

```bash
mkdir -p ~/.config/sandboxed-agents
```

### Add a bind for all agents

```bash
# ~/.config/sandboxed-agents/common.conf
USER_BINDS+=(
  --ro-bind-try "$HOME/.cargo" "$HOME/.cargo"
)
```

### Disable SSH/GPG for a specific agent

```bash
# ~/.config/sandboxed-agents/codex.conf
ALLOW_SSH=false
ALLOW_GPG=false
```

`ALLOW_SSH` and `ALLOW_GPG` are feature flags (default `true`). They can be
set in any config layer — common applies to all agents, per-agent limits to one.

### Replace an agent's full bind set

Use `=` instead of `+=`:

```bash
# ~/.config/sandboxed-agents/omp.conf
AGENT_BINDS=(
  --bind-try "$HOME/.omp" "$HOME/.omp"
  --bind-try "$HOME/.pi" "$HOME/.pi"
)
```

## Add a new agent

1. Create a profile in `config/`:

```bash
cat > config/qwen.conf << 'CONF'
AGENT_BIN_DEFAULT=/usr/bin/qwen
AGENT_LABEL="Qwen"
AGENT_BINDS=(
  --bind-try "$HOME/.qwen" "$HOME/.qwen"
)
CONF
```

2. (Optional) Create a symlink for convenience:

```bash
ln -s agent-wrapper qwen
./qwen
```

## Config reference

| Variable | Default | Purpose |
|---|---|---|
| `AGENT_BIN_DEFAULT` | — | Binary path for the agent |
| `AGENT_LABEL` | `$AGENT_NAME` | Display name in error messages |
| `ROOTFS_BINDS` | (bwrap) | Read-only root filesystem mounts |
| `ETC_BINDS` | (bwrap) | Read-only system config mounts |
| `USER_BINDS` | (bwrap) | User home directory mounts |
| `WORK_BINDS` | (bwrap) | Working directory mount |
| `AGENT_BINDS` | — | Agent-specific mounts |
| `ENVS` | HOME, USER | Environment variables |
| `EXTRA_ARGS` | — | Extra bwrap arguments |
| `ALLOW_SSH` | `true` | Allow SSH agent socket bind |
| `ALLOW_GPG` | `true` | Allow GPG agent bind |