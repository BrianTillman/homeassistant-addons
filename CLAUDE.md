# Home Assistant Addon Development Guide

This document captures lessons learned from developing and testing local Home Assistant addons in the devcontainer environment.

## Environment Overview

### Devcontainer Architecture
- **Host machine**: Your local development environment (e.g., `btillman@dev`)
- **HA Devcontainer**: `ghcr.io/home-assistant/devcontainer:addons` running HA Supervisor
- **SSH access**: Connect via `ssh -p 2255 root@localhost` (appears as `[local-ssh ~]$`)
- **HA Web UI**: http://localhost:7123
- **Addon source**: Mounted at `/addons/homeassistant-addons/` inside devcontainer

### Key Paths
| Path | Description |
|------|-------------|
| `/addons/homeassistant-addons/` | Addon source code (mounted from host) |
| `/config/` | Home Assistant configuration directory |
| `/data/addons/data/<slug>/` | Addon persistent data |
| `/tmp/` | Temporary files (backup locations, etc.) |

## Critical: Image vs Local Build

### The Problem
If `config.yaml` contains an `image:` line, the addon pulls a pre-built image from Docker Hub instead of building from local source:

```yaml
# This causes the addon to IGNORE your local code!
image: homeassistant/{arch}-addon-git_pull
```

### The Solution
Comment out or remove the `image:` line to build from local Dockerfile:

```yaml
#image: homeassistant/{arch}-addon-git_pull
```

### Verification
After changing config.yaml:
```bash
ha addons info local_<slug> | grep "build:"
# Should show: build: true
```

If it still shows `build: false`, you need to:
1. Uninstall and reinstall the addon, OR
2. Rebuild the devcontainer

## Addon Management Commands

### Inside devcontainer (local-ssh)

```bash
# List all addons
ha addons list

# Install addon
ha addons install local_git_pull

# Uninstall addon
ha addons uninstall local_git_pull

# Rebuild addon (only works if build: true)
ha addons rebuild local_git_pull

# Start/stop addon
ha addons start local_git_pull
ha addons stop local_git_pull

# View logs
ha addons logs local_git_pull

# Get addon info
ha addons info local_git_pull

# Set addon options
ha addons options local_git_pull --options key="value"

# Reload supervisor (use sparingly)
ha supervisor reload

# Restart supervisor (will disconnect SSH!)
ha supervisor restart
```

### Addon Slug Naming
- Local addons use `local_<slug>` format
- The slug comes from `config.yaml`'s `slug:` field
- Example: `slug: git_pull` → `local_git_pull`

## Rebuilding Addons

### When `ha addons rebuild` fails with "Can't rebuild an image based add-on"

This means the supervisor cached the addon as image-based. Fix:

```bash
# Option 1: Uninstall/reinstall
ha addons uninstall local_git_pull
ha addons install local_git_pull

# Option 2: If that doesn't work, restart supervisor
ha supervisor restart
# Wait for reconnection, then reinstall

# Option 3: Nuclear option - rebuild devcontainer
# From host machine, rebuild the devcontainer without cache
```

### Verifying Your Code is Running

After rebuild, verify the actual code in the image:

```bash
# From HOST machine (not devcontainer)
docker images | grep git_pull
docker run --rm --entrypoint cat <image_name> /run.sh | grep -A5 "your search term"
```

Note: Addon containers exit immediately after running, so you can't `docker exec` into them.

## Git Repository Ownership

Inside devcontainer, you may encounter:
```
fatal: detected dubious ownership in repository at '/addons/homeassistant-addons'
```

Fix:
```bash
git config --global --add safe.directory /addons/homeassistant-addons
```

## Testing Workflow

### Test Setup for git_pull Addon

1. **Ensure `/config` is NOT a git repo** (to trigger fresh clone path):
   ```bash
   rm -rf /config/.git
   ```

2. **Create local test files** (should be preserved after clone):
   ```bash
   mkdir -p /config/deps
   echo "test_data" > /config/deps/test_dep.txt
   touch /config/home-assistant_v2.db
   echo "local_secret: my_value" > /config/secrets.yaml
   ```

3. **Configure addon**:
   ```bash
   ha addons options local_git_pull --options repository="https://github.com/USER/REPO.git" --options git_branch="master"
   ```

4. **Run and check logs**:
   ```bash
   ha addons start local_git_pull
   ha addons logs local_git_pull
   ```

5. **Verify results**:
   ```bash
   ls -la /config/.git/           # Git repo created
   ls -la /config/deps/           # Local files preserved
   cat /config/secrets.yaml       # Local secrets preserved
   ```

## Bash Scripting Pitfalls

### Extglob Parse-Time Error

**Problem**: Bash parses the entire script before execution. Extglob patterns like `!(*.yaml)` cause syntax errors even if `shopt -s extglob` appears before them in the code.

**Broken**:
```bash
function my-func {
    shopt -s extglob
    cp -r /src/!(*.yaml) /dest    # ERROR: parsed before shopt runs!
    shopt -u extglob
}
```

**Fixed** (use eval to delay parsing):
```bash
function my-func {
    shopt -s extglob nullglob dotglob
    eval 'cp -r /src/!(*.yaml|*.yml) /dest 2>/dev/null' || true
    shopt -u extglob nullglob dotglob
}
```

### Unbound Variable Errors

With `set -u` (or `set -o nounset`), referencing unset variables causes errors.

**Problem**:
```bash
function git-clone {
    # Does NOT set OLD_COMMIT
}

function validate-config {
    if [ "$OLD_COMMIT" == "$NEW_COMMIT" ]; then  # ERROR: OLD_COMMIT unbound!
```

**Fix**: Initialize variables before conditional paths:
```bash
function git-synchronize {
    OLD_COMMIT=""  # Initialize early

    if ! git rev-parse --is-inside-work-tree &>/dev/null; then
        git-clone
        return  # OLD_COMMIT stays empty
    fi

    OLD_COMMIT=$(git rev-parse HEAD)  # Set if repo exists
}
```

## Test Fixtures Repository

For git_pull addon testing, create a test repo with:

### Files that should come FROM GIT (YAML/YML):
- `configuration.yaml`
- `automations.yaml`
- `scripts.yaml`
- `test_yml.yml` (tests .yml handling)
- `secrets.yaml` (placeholder - local version should override)

### Files that should be PRESERVED from local:
- `deps/` - directory not in git
- `home-assistant_v2.db` - database files
- `.storage/` - hidden directories
- `custom_components/` - custom integrations

### Test Verification Matrix

| File | Source | Verification |
|------|--------|--------------|
| `configuration.yaml` | Git | Content matches repo |
| `secrets.yaml` | Local | Content is local value, not repo placeholder |
| `deps/` | Local | Directory exists after clone |
| `*.db` files | Local | Files exist after clone |
| `.yml` files | Git | Not restored from backup |

## Debugging Tips

### Check addon source code inside devcontainer
```bash
cat /addons/homeassistant-addons/<addon>/data/run.sh | grep -A5 "search term"
```

### Check built image code (from host)
```bash
docker run --rm --entrypoint cat <image>:<tag> /run.sh | grep -A5 "search term"
```

### View backup contents (git_pull)
Backups are created at `/tmp/config-YYYY-MM-DD_HH-MM-SS/`:
```bash
ls -la /tmp/config-*/
```

### Common Log Messages

| Message | Meaning |
|---------|---------|
| `[Warn] Git repository doesn't exist` | Fresh clone path triggered |
| `[Info] Local git repository exists` | Normal pull/reset path |
| `Fresh clone detected, validating configuration...` | OLD_COMMIT fix working |
| `syntax error near unexpected token` | Extglob parse-time error |
| `unbound variable` | Variable not initialized |

## PR Workflow

When creating PRs for Home Assistant addons:

1. **Test locally first** using the devcontainer
2. **Update CHANGELOG.md** with version bump
3. **Keep commits focused** - one fix per commit or squash related changes
4. **Include test plan** in PR description
5. **Don't push to upstream** - always target your fork first
