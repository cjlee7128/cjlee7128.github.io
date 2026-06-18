---
layout: post
title: "Running MLModelScope Locally with the PyTorch Agent"
date: 2026-06-18 11:43:00 -0400
category: tutorials
---

This guide walks you through running the three core components of **MLModelScope** locally for development and testing:

* **`mlmodelscope`**: The React-based frontend.
* **`mlmodelscope-api`**: The backend services, including the API, database, RabbitMQ, tracing, and upload helper.
* **`py-mlmodelscope`**: The local PyTorch model agent.

---

## Prerequisites

Before starting, ensure your host machine has the following tools installed:

* **Git**
* **Docker & Docker Compose**
* **Node.js `14.21.3` and npm `6.14.18`**
* **Python or Conda** (only needed if running the Python API directly for debugging)

You can use a Node version manager like `nvm` to select the correct Node.js version:

```bash
nvm install 14.21.3
nvm use 14.21.3
node -v # Expected: v14.21.3
npm -v  # Expected: 6.14.18
```

---

## 1. Repository Layout

Create a dedicated workspace directory and clone the three required repositories:

```bash
mkdir -p ~/mlmodelscope2
cd ~/mlmodelscope2

git clone https://github.com/c3sr/mlmodelscope
git clone https://github.com/c3sr/mlmodelscope-api
git clone https://github.com/xlab-ub/py-mlmodelscope
```

The instructions below assume you are working from within this layout.

---

## 2. Configure the API Stack

### Database & RabbitMQ Environment
Create the `.env` file for the API backend:

```bash
cd ~/mlmodelscope2/mlmodelscope-api
cp .env.example .env 2>/dev/null || touch .env
```

Edit `mlmodelscope-api/.env` and replace the angle-bracket placeholders with your local environment values (do not commit this file to git):

```dotenv
DOCKER_REGISTRY=local-registry
ENVIRONMENT=local.
API_VERSION=latest

DB_DRIVER=postgres
DB_HOST=localhost
DB_PORT=15432
DB_USER=<local-db-user>
DB_PASSWORD=<local-db-password>
DB_DBNAME=<local-db-name>

MQ_HOST=localhost
MQ_PORT=5672
MQ_USER=<local-mq-user>
MQ_PASSWORD=<local-mq-password>
MQ_ERLANG_COOKIE=<local-rabbitmq-cookie>

TRACER_ADDRESS=localhost:6831
```

### Local Upload Companion Environment
Create `.env.companion` for local model upload support:

```bash
cd ~/mlmodelscope2/mlmodelscope-api
touch .env.companion
```

Populate the companion environment file with mock AWS credentials for local storage:

```dotenv
COMPANION_AWS_KEY=<local-placeholder-key>
COMPANION_AWS_SECRET=<local-placeholder-secret>
COMPANION_AWS_BUCKET=<local-placeholder-bucket>
COMPANION_AWS_REGION=us-east-1
```

> [!WARNING]
> Never put production AWS credentials or keys inside local configuration files.

---

## 3. Configure the Frontend

Create the `.env` file for the React application:

```bash
cd ~/mlmodelscope2/mlmodelscope
cp .env.example .env
```

Set the local development API endpoints:

```dotenv
REACT_APP_API_URL=http://localhost:8005
REACT_APP_COMPANION_URL=http://localhost
REACT_APP_IS_LOCAL=true
```

Install the project dependencies:

```bash
cd ~/mlmodelscope2/mlmodelscope
npm install
```

---

## 4. Start the API Stack

Run Docker Compose from the `mlmodelscope-api` directory to start the database, message broker (RabbitMQ), tracer (Jaeger), and upload services:

```bash
cd ~/mlmodelscope2/mlmodelscope-api
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d --build
```

### Useful Local Endpoints
* **API Service**: [http://localhost:8005](http://localhost:8005)
* **Frontend Web App**: [http://localhost:3000](http://localhost:3000)
* **RabbitMQ Management Console**: [http://localhost:15672](http://localhost:15672)
* **Jaeger Tracing UI**: [http://localhost:16686](http://localhost:16686)

You can check the container status and inspect the backend logs using:

```bash
docker compose ps
docker compose logs -f api
```

---

## 5. Optional: Run the Python API Directly

If you need to debug or step through `python_api` directly on your host machine instead of running it inside Docker, set up a Conda environment:

```bash
cd ~/mlmodelscope2/mlmodelscope-api/python_api
conda create -n mlms-api python=3.8 -y
conda activate mlms-api
pip install -r requirements.txt
```

Export the database and message broker settings to your shell:

```bash
export DB_HOST=localhost
export DB_PORT=15432
export DB_USER=<local-db-user>
export DB_PASS=<local-db-password>
export DB_NAME=<local-db-name>

export MQ_HOST=localhost
export MQ_PORT=5672
export MQ_USER=<local-mq-user>
export MQ_PASS=<local-mq-password>
```

Start the API server on port `8005`:

```bash
fastapi run api.py --reload --port 8005
```

---

## 6. Build the PyTorch Agent

To run models locally, you will need the PyTorch CPU execution agent. The local agent should copy your local workspace checkout rather than cloning the remote repository.

Copy the default Dockerfile to a temporary location:

```bash
cd ~/mlmodelscope2
cp py-mlmodelscope/dockerfiles/pytorch/Dockerfile.cpu_pytorch2.0.1 /tmp/Dockerfile.cpu_pytorch-local
```

Open `/tmp/Dockerfile.cpu_pytorch-local` in an editor, comment out or replace the remote Git clone line, and add a `COPY` instruction instead:

```dockerfile
# Replace the remote clone line:
# RUN git clone https://github.com/xlab-ub/py-mlmodelscope.git /py-mlmodelscope

# With a local copy from your host machine:
COPY py-mlmodelscope /py-mlmodelscope
```

Build the local Docker image:

```bash
cd ~/mlmodelscope2
docker build \
  -t pytorch-agent:local \
  -f /tmp/Dockerfile.cpu_pytorch-local \
  .
```

---

## 7. Run the PyTorch Agent

Launch the PyTorch agent container (ensure the API stack and RabbitMQ are already running):

```bash
cd ~/mlmodelscope2/mlmodelscope-api

# Stop any existing agent container
docker rm -f pytorch-agent 2>/dev/null || true

# Run the CPU agent
docker run -d \
  --name pytorch-agent \
  --network host \
  --shm-size=1g \
  --ulimit memlock=-1 \
  --ulimit stack=67108864 \
  --env-file .env \
  pytorch-agent:local
```

Follow the agent log output to confirm it successfully registers and subscribes to the queue:

```bash
docker logs -f pytorch-agent
```

The agent should start consuming tasks from the PyTorch queue: `agent-pytorch-amd64`.

---

## 8. Run the Frontend & Test a Model

Launch the React application:

```bash
cd ~/mlmodelscope2/mlmodelscope
npm run start
```

Open your browser to [http://localhost:3000](http://localhost:3000) and follow these steps to run a prediction task:

1. Select a PyTorch torchvision **ResNet** model (e.g., `resnet50`).
2. Upload an image for classification.
3. (Optional) Toggle **Explain this prediction** to trigger the Explainable AI (XAI) routine.
4. Click **Run model and see results**.

---

## Troubleshooting

### No models are visible in the frontend
Ensure the API container can connect to the PostgreSQL database. Check the container statuses and API logs:
```bash
docker compose ps
docker compose logs -f api
```

### The prediction trial hangs in "pending"
This indicates the PyTorch agent is not receiving the task from RabbitMQ. Check that RabbitMQ is running and confirm the agent logs show active polling:
```bash
docker compose logs -f mq
docker logs -f pytorch-agent
```

### The agent cannot connect to RabbitMQ
* On Linux, `--network host` allows the container to connect via `localhost`.
* On Docker Desktop (macOS/Windows), you may need to update `MQ_HOST` in `.env` to `host.docker.internal` so the container can resolve services running on the host machine.
* Verify that your port mappings match (RabbitMQ: `5672`, PostgreSQL: `15432`, Frontend: `3000`, API: `8005`).
