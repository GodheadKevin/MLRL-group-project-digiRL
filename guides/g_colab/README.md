# Remote Google Colab Development & Synchronization Guide

This document describes the workflow for developing locally (e.g., using Neovim, VS Code, or your preferred IDE) while executing code and workloads remotely on a Google Colab GPU instance. It covers SSH proxy configuration, real-time file synchronization via `fswatch` and `rsync`, and setting up isolated Python virtual environments with `uv`.

---

## 1. Local Machine Prerequisites

Ensure your local development machine has the required command-line tools installed.

### macOS (Homebrew)
```bash
brew install fswatch rsync
```

### Linux (Debian / Ubuntu)
```bash
sudo apt-get update && sudo apt-get install -y fswatch rsync
```

### SSH Key Setup
Generate an SSH key pair on your local machine if you do not already have one:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```

---

## 2. Launching Colab & Enabling SSH via CLI

Google Colab instances do not possess public IP addresses or open inbound ports. Access is routed through an encrypted tunnel handled by the `colab` CLI.

1. **Authenticate the Colab CLI** (run once locally):
   ```bash
   colab auth login
   ```

2. **Create a remote session** (example: named `digirl_1` with GPU acceleration):
   ```bash
   colab session create --name digirl_1 --accelerator gpu
   ```

3. **Check session status**:
   ```bash
   colab session list
   ```

---

## 3. Configuring OpenSSH (`~/.ssh/config`)

To allow standard tools (`ssh`, `rsync`, `scp`, editor remotes) to connect to your Colab session without managing WebSocket connections manually, configure a `ProxyCommand`.

Add the following block to your local `~/.ssh/config`:

```ssh-config
Host colab-digirl
    HostName localhost
    User root
    IdentityFile ~/.ssh/id_ed25519
    ProxyCommand colab ssh -s digirl_1 --proxy-mode -i ~/.ssh/id_ed25519
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    ServerAliveInterval 60
    ServerAliveCountMax 5
```

### Testing Direct SSH Access
Connect to the Colab VM directly from your local terminal:
```bash
ssh colab-digirl
```

Verify GPU availability and shell path inside Colab:
```bash
nvidia-smi
pwd
exit
```

---

## 4. Remote Python Environment Setup Using `uv`

Inside your SSH session, use `uv` to maintain a standalone Python virtual environment pinned to a specific version (e.g., Python 3.10) without inheriting host system packages.

1. **Connect to the Colab instance**:
   ```bash
   ssh colab-digirl
   ```

2. **Install `uv`**:
   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   source $HOME/.cargo/env
   ```

3. **Create the project directory**:
   ```bash
   mkdir -p /content/MLRL-group-project-digiRL
   cd /content/MLRL-group-project-digiRL
   ```

4. **Create a clean virtual environment**:
   ```bash
   # Clear any system-injected PYTHONPATH
   unset PYTHONPATH

   # uv will fetch standalone Python 3.10 if not present on the VM
   uv venv --python 3.10 .venv
   ```

5. **Activate the environment**:
   ```bash
   source .venv/bin/activate
   ```

6. **Verify environment isolation**:
   ```bash
   python --version           # Should output: Python 3.10.x
   which python              # Points to: /content/MLRL-group-project-digiRL/.venv/bin/python
   cat .venv/pyvenv.cfg      # Shows: include-system-site-packages = false
   ```

7. **Install dependencies**:
   ```bash
   uv pip install -r requirements.txt
   uv pip install -e .
   ```

---

## 5. Automated Local-to-Remote File Synchronization

Keep your remote code directory continuously up-to-date with your local working copy using an `rsync` exclusion filter and an `fswatch` loop.

### Step 5.1: Create `.rsyncignore`
In your local project repository root, create `.rsyncignore` to avoid transferring massive binaries, caches, and environment directories:

```text
.git/
.venv/
.digirl-env/
__pycache__/
*.pyc
*.pt
*.bin
*.safetensors
*.zip
*.tar.gz
checkpoints/
Auto-UI-Base/
```

### Step 5.2: Create `sync_to_colab.sh`
In your local repository root, create `sync_to_colab.sh`:

```bash
#!/usr/bin/env bash

set -euo pipefail

LOCAL_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/"
REMOTE_HOST="colab-digirl"
REMOTE_DIR="/content/MLRL-group-project-digiRL/"
EXCLUDE_FILE="$LOCAL_DIR.rsyncignore"

echo "==> Performing initial synchronization to $REMOTE_HOST:$REMOTE_DIR..."
rsync -avz --delete --exclude-from="$EXCLUDE_FILE" "$LOCAL_DIR" "$REMOTE_HOST:$REMOTE_DIR"

echo "==> Monitoring local changes with fswatch..."
fswatch -o -e "\.git" -e "\.venv" -e "__pycache__" "$LOCAL_DIR" | while read -r event; do
    echo "[$(date '+%H:%M:%S')] Fi[118;1:3ule change detected. Syncing..."
    rsync -avz --exclude-from="$EXCLUDE_FILE" "$LOCAL_DIR" "$REMOTE_HOST:$REMOTE_DIR"
done
```

### Step 5.3: Run the Synchronizer
Make the script executable and run it in a background or separate terminal session:
```bash
chmod +x sync_to_colab.sh
./sync_to_colab.sh
```

Any code changes saved in your local editor will immediately sync to `/content/MLRL-group-project-digiRL/` on the remote Colab instance while preserving remote model weights and virtual environment files.

