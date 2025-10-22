#  Self-Hosted GitHub Actions Runner Setup

## Overview
This project demonstrates how to set up a **self-hosted GitHub Actions runner** on a local environment (Ubuntu on WSL).  
It automates the process using a Bash script, which downloads, registers, and installs the runner as a service.

# Step-by-Step Setup Process


## Requirements

- A GitHub repository  
- Personal Access Token (classic) with the following scopes:  
  - `repo`  
  - `workflow`  
  - `admin:repo_hook`  
- A Linux, Windows, or Mac environment (in my case: Ubuntu on WSL)



## Script Used: `setup_github_runner.sh`

```sh

set -eu

echo "=== GitHub Self-Hosted Runner Setup ==="

# Ask for required information
printf "Enter your GitHub repository (format: owner/repo): "
read REPO
printf "Enter your GitHub Personal Access Token (PAT): "
stty -echo
read GITHUB_PAT
stty echo
printf "\n"

# Validate input
if [ -z "$REPO" ] || [ -z "$GITHUB_PAT" ]; then
    echo "[ERROR] Repository and token are required."
    exit 1
fi

# Variables
RUNNER_DIR="$HOME/github-runner"
RUNNER_VERSION="2.320.0"
RUNNER_TGZ="actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"
RUNNER_URL="https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/${RUNNER_TGZ}"
RUNNER_NAME="$(hostname)"
API_URL="https://api.github.com/repos/${REPO}/actions/runners/registration-token"

# Ensure curl exists
if ! command -v curl >/dev/null 2>&1; then
    echo "[INFO] Installing curl..."
    sudo apt-get update -y && sudo apt-get install -y curl
fi

# Create runner directory
mkdir -p "$RUNNER_DIR"
cd "$RUNNER_DIR"

# Download runner package if missing
if [ ! -f "$RUNNER_TGZ" ]; then
    echo "[INFO] Downloading GitHub Actions Runner v${RUNNER_VERSION}..."
    curl -L -O "$RUNNER_URL"
else
    echo "[INFO] Runner package already exists. Skipping download."
fi

# Extract runner package
echo "[INFO] Extracting runner package..."
tar xzf "$RUNNER_TGZ"

# Get registration token
echo "[INFO] Requesting registration token..."
TOKEN_JSON=$(curl -s -X POST -H "Authorization: token ${GITHUB_PAT}" "$API_URL")
REG_TOKEN=$(echo "$TOKEN_JSON" | grep -o '"token": *"[^"]*"' | cut -d '"' -f4)

if [ -z "$REG_TOKEN" ]; then
    echo "[ERROR] Failed to get registration token. Please check your PAT scopes and repository name."
    exit 1
fi

# Configure the runner
echo "[INFO] Configuring GitHub Actions runner..."
./config.sh --url "https://github.com/${REPO}" \
            --token "$REG_TOKEN" \
            --name "$RUNNER_NAME" \
            --unattended \
            --replace

# Install and start as a service
echo "[INFO] Installing runner as a service..."
sudo ./svc.sh install

echo "[INFO] Starting the runner service..."
sudo ./svc.sh start

echo "=== Setup complete! ==="
echo "Runner successfully registered for repository: ${REPO}"
```


##  Steps to Run

1. Save the script above as **`setup_github_runner.sh`**
2. Make it executable:
   ```bash
   chmod +x setup_github_runner.sh

###  Run the Script

Once everything is set up, run the script:

```bash
./setup_github_runner.sh



It will prompt to Enter the following:

- **Repository** → e.g. `Kelkaal/Local_Runner`  
- **GitHub Token** → your personal access token  

---

### The script will automatically:

1. **Download** the GitHub runner package  
2. **Register** it with your repository  
3. **Install and start** it as a system service

## Now Run a Test Pipeline Using the Self-Hosted Runner:

Create a workflow in your repository:
  .github/workflows/test-runner.yml


### Expected Result

Once you push this workflow to your repository:

- Your **local runner** (running under WSL) will automatically **pick up the job**.  
- You’ll see the job running in **GitHub → Actions → Test Self-Hosted Runner**.

  <img width="1769" height="603" alt="Screenshot 2025-10-22 003321" src="https://github.com/user-attachments/assets/31f53114-62cf-4ce5-94b0-23f3bf3e9c32" />






## Challenges Faced & How I Solved Them

| **Challenge** | **Description** | **Solution** |
|----------------|-----------------|---------------|
| **403 Forbidden Error** | I initially got a 403 error when fetching the registration token using my script. | Realized my Personal Access Token lacked the right scopes. Recreated a classic token with **repo**, **workflow**, and **admin:repo_hook** permissions. |
| **Incorrect Repo Format** | The repo was initially entered as a full URL (`https://github.com/Kelkaal/local_L_runner.git`) instead of the required `owner/repo` format. | Fixed it by using only `Kelkaal/local_L_runner`. |
| **Network Issues on WSL** | The runner occasionally failed to start due to WSL network settings. | Restarted WSL and confirmed connectivity before starting the service. |

 
 ## What I’d Do Differently in Production

-  **Use Docker** to isolate the runner environment.  
-  **Run it on a dedicated server or EC2 instance** instead of a local WSL machine.  
-  **Add monitoring and alerting** for runner failures.  
-  **Use `systemd` or container restart policies** for automatic recovery.  
-  **Store tokens securely** using GitHub Secrets, not inline in scripts.

## Security Considerations Implemented

-  **Token input is hidden** using `stty -echo` to prevent it from displaying in the terminal.  
-  **Script validates** that both `REPO` and `PAT` are provided before continuing.  
- **No token is logged or echoed** back to the terminal or stored in plain text.  
-  **Runner operates as a non-root user**, except when elevated privileges are required for service setup.  
-  **Token scopes minimized** to only what’s necessary — `repo` and `workflow`.
  







