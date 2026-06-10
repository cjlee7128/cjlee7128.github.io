---
layout: post
title: "Hosting LLMs via vLLM on Tenstorrent LoudBox"
date: 2026-06-09 23:15:00 -0400
category: tutorials
---

This guide walks you through setting up and hosting Large Language Models (LLMs) like `Llama-3.3-70B-Instruct` and `gpt-oss-120b` on a **Tenstorrent LoudBox** system using vLLM and the Tenstorrent inference server stack.

---

## 1. Environment Setup

### Anaconda Environment
First, create and activate a clean Conda environment running Python 3.12:

```bash
conda create -n ttloudbox python=3.12 -y
conda activate ttloudbox
```

### Install Tenstorrent System Software
Update your local package lists and run the official installer script to set up the driver and metalium stack:

```bash
sudo apt update && sudo apt install -y curl jq
/bin/bash -c "$(curl -fsSL https://github.com/tenstorrent/tt-installer/releases/latest/download/install.sh)"
```

To verify the system software installation, activate the virtual environment and query the device status:

```bash
source ~/.tenstorrent-venv/bin/activate
tt-smi
```

---

## 2. Docker Installation & Post-Install Setup

Docker is required to run the inference server container. If it is already installed, you can skip this step.

### Clean and Setup Repository
Uninstall any old Docker packages:

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
```

Add Docker's official GPG key and repository sources:

```bash
# Add official GPG key
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add repository to APT sources
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

### Install Docker Engine
Install Docker components:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify that the service is running:

```bash
sudo systemctl status docker
sudo docker run hello-world
```

### Manage Docker as a Non-Root User
Ensure you can run Docker commands without prepending `sudo`:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

---

## 3. Host System Tuning (Optional)

For peak inference performance, it is recommended to set the CPU frequency scaling governor to `performance` mode:

```bash
sudo apt-get update && sudo apt-get install -y linux-tools-generic
sudo cpupower frequency-set -g performance
```

> [!NOTE]
> If you encounter a warning saying `cpupower` is not found for your kernel version (e.g., `5.15.0-173`), run:
> `sudo apt-get install -y linux-tools-5.15.0-173-generic` (replacing the version number with your specific kernel) and retry.

---

## 4. Configuring Model Access & Inference Server

### Export Hugging Face Token
To download gated or private models (like Llama 3.3), export your Hugging Face API token:

```bash
export HF_TOKEN="your_hugging_face_access_token_here"
```

### Select Your Hardware Device
Select your device and model environment variables. For a **TT-LoudBox (Wormhole)** system, run this interactive selector script and choose option **3**:

```bash
select_device_and_model(){ echo -e "\nSelect a Tenstorrent system from the list below:"; PS3=$'\n#? '; options=("TT-QuietBox (Wormhole)" "TT-QuietBox (Blackhole)" "TT-LoudBox (Wormhole)" "TT-LoudBox (Blackhole)" "n150s" "n150d" "n300s" "n300d" "p100a" "p150a" "p150b" "Quit"); select opt in "${options[@]}"; do case "$opt" in "TT-QuietBox (Wormhole)") DEVICE="t3k"; MODEL="Llama-3.3-70B-Instruct";; "TT-QuietBox (Blackhole)") DEVICE="p150x4"; MODEL="Llama-3.3-70B-Instruct";; "TT-LoudBox (Wormhole)") DEVICE="t3k"; MODEL="Llama-3.3-70B-Instruct";; "TT-LoudBox (Blackhole)") DEVICE="p150x8"; MODEL="Llama-3.3-70B-Instruct";; "n150s"|"n150d") DEVICE="n150"; MODEL="Llama-3.1-8B-Instruct";; "n300s"|"n300d") DEVICE="n300"; MODEL="Llama-3.1-8B-Instruct";; "p100a") DEVICE="p100"; MODEL="Llama-3.1-8B-Instruct";; "p150a"|"p150b") DEVICE="p150"; MODEL="Llama-3.1-8B-Instruct";; "Quit") echo "❌ Exiting without setting any environment variables."; return;; *) echo "❌ Invalid option. Try again."; continue; esac; export DEVICE MODEL; echo -e "\n✅ DEVICE set to '$DEVICE'"; echo "✅ MODEL set to '$MODEL'"; break; done; }; select_device_and_model
```

### Verify Hugging Face Access
Verify that you have read access to the target model repository using this snippet:

```bash
check_hf_access() { [ -z "$MODEL" ] && { printf "✖ Error: Please provide a Hugging Face repository ID.\n"; return 1; }; ! command -v curl &>/dev/null && { printf "✖ Error: curl is not installed.\n"; return 1; }; local REPO_ID="meta-llama/$MODEL"; local TOKEN=${HF_TOKEN:-$(cat "$HOME/.cache/huggingface/token" 2>/dev/null)}; [ -z "$TOKEN" ] && printf "ℹ️ Info: No Hugging Face token found.\n   You can only access public repositories.\n"; local AUTH_HEADER=""; [ -n "$TOKEN" ] && AUTH_HEADER="Authorization: Bearer $TOKEN"; printf "Checking access for: %s...\n" "$REPO_ID"; local URL="https://huggingface.co/$REPO_ID/resolve/main/config.json"; local HTTP_CODE=$(curl -s -L -o /dev/null -w "%{http_code}" -H "$AUTH_HEADER" "$URL"); case $HTTP_CODE in 200) printf "✔ Access granted.\n";; 401) printf "✖ Access denied (401 Unauthorized).\n  This is a private or gated repository.\n  Ensure your token is valid and has the correct permissions.\n";; 403) printf "✖ Access forbidden (403 Forbidden).\n  The repository is gated.\n  You need to visit the repository page on Hugging Face and request access.\n";; 404) printf "✖ Repository or 'config.json' not found (404 Not Found).\n  Please check if the repository ID '$REPO_ID' is correct.\n";; *) printf "✖ Failed to check access.\n  Received HTTP status code: %s\n" "$HTTP_CODE";; esac; }; HF_HUB_DISABLE_XET=1; check_hf_access;
```

### Clone the Inference Server Repo
Clone the repository and checkout the latest stable release tag:

```bash
git clone https://github.com/tenstorrent/tt-inference-server.git
cd tt-inference-server
git fetch --tags
git checkout $(git tag -l "v*" --sort=-v:refname | head -n 1)
```

---

## 5. Setting Up Mesh Topology

Tenstorrent systems rely on dynamic device interconnects. For multi-device execution (like LoudBox), you must set up the mesh topology:

```bash
git clone https://github.com/tenstorrent/tt-topology.git
cd tt-topology
pip install .
tt-topology -l mesh -p mesh_layout.png
```

> [!WARNING]
> If you omit this step, launching the server might fail with:
> `ValueError: ModelSpec requires a system-level topology of SystemTopology.MESH, detected SystemTopology.ISOLATED`.

---

## 6. Running the Inference Server

To launch the server, activate the Tenstorrent Python virtual environment, select your model, and run the server orchestration script.

### Launching Llama-3.3-70B-Instruct
Launch the Llama 3.3 server:

```bash
# cd tt-inference-server
source ~/.tenstorrent-venv/bin/activate
python3 run.py \
  --model Llama-3.3-70B-Instruct \
  --device t3k \
  --workflow server \
  --docker-server \
  --service-port 8090 \
  --no-auth
```

### Launching gpt-oss-120b
Alternatively, to launch `gpt-oss-120b`:

```bash
# cd tt-inference-server
source ~/.tenstorrent-venv/bin/activate
python3 run.py \
  --model gpt-oss-120b \
  --device t3k \
  --workflow server \
  --docker-server \
  --service-port 8090 \
  --no-auth
```

> [!NOTE]
> The first initialization of a large model may take up to **10 minutes** to set up, download weights, and compile. Follow the CLI prompts to select Hugging Face as the weight provider when asked.

---

## 7. Verifying Server Status & Querying

Once the server is running, check its health:

```bash
check_server_health(){ code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8090/health); exit_code=$?; if [[ $exit_code -ne 0 ]]; then echo "❌ Error: Unable to connect to server at localhost:8090"; elif [[ $code -eq 200 ]]; then echo "✅ Server is ready (HTTP 200)"; else echo "⚠️ Server responded with status: $code"; fi; }; check_server_health
```

### Sending an Example Request
Send a test inference request to the `/v1/completions` endpoint:

```bash
curl -sS "http://localhost:8090/v1/completions" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"meta-llama/Llama-3.3-70B-Instruct\",
    \"prompt\": \"San Francisco is a\",
    \"max_tokens\": 50,
    \"temperature\": 0
  }" | jq
```

Or for `gpt-oss-120b` via the chat completion endpoint:

```bash
curl -s http://localhost:8090/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-oss-120b",
    "messages": [{"role": "user", "content": "San Francisco is a"}],
    "max_tokens": 512,
    "temperature": 0,
    "reasoning_effort": "low",
    "include_reasoning": true
  }' | jq '.choices[0].message'
```

---

## 8. Troubleshooting Tips

* **Docker container keeps crashing**: Try running `sudo reboot` and resetting the hardware links using `tt-smi -r`.
* **VS Code Terminal Issues**: If Docker-related errors lock up your editor or terminal interface, clean up the node processes by running `killall -9 node` and reloading your editor window.
