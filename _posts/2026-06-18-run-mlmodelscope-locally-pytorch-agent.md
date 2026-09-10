---
layout: post
title: "Running MLModelScope Locally with the PyTorch Agent"
date: 2026-06-18 11:43:00 -0400
category: tutorials
---

This guide walks you through running the four core components of **MLModelScope** locally for development, testing, and Explainable AI (XAI):

* **`mlmodelscope`**: The React-based frontend web application.
* **`mlmodelscope/explanation-api`**: The AI explanation service providing multimodal model insights.
* **`mlmodelscope-api`**: The backend services, including the frontend API, database, RabbitMQ, tracing, and upload helper.
* **`py-mlmodelscope`**: The local PyTorch model agent.

The instructions below are intentionally local-only. Replace placeholder values like `<local-db-password>` with machine-specific credentials, and keep all `.env` files untracked.

---

## Prerequisites

Ensure your host machine has the following dependencies installed:

* **Git**
* **Docker & Docker Compose**
* **Node.js 20 or newer** (Node.js `22.23.2` and npm `10.9.8` are pinned for the frontend and explanation API)
* **Python or Conda** (only needed if running `python_api` directly for debugging)

You can use `nvm` to install and activate the pinned Node.js runtime:

```bash
nvm install 22.23.2
nvm use 22.23.2
node -v # Expected: v22.23.2
npm -v  # Expected: 10.9.8
```

---

## 1. Repository Layout

Create a dedicated workspace directory and clone the three required repositories:

```bash
mkdir -p ~/mlmodelscope2
cd ~/mlmodelscope2

git clone https://github.com/xlab-ub/mlmodelscope
git clone https://github.com/xlab-ub/mlmodelscope-api
git clone https://github.com/xlab-ub/py-mlmodelscope
```

The subsequent steps assume this directory structure.

---

## 2. Configure the API Stack

### Database & RabbitMQ Environment
Create the `.env` file for the API backend:

```bash
cd ~/mlmodelscope2/mlmodelscope-api
cp .env.example .env 2>/dev/null || touch .env
```

Edit `mlmodelscope-api/.env` with local configuration values:

```dotenv
DOCKER_REGISTRY=<docker-registry-or-namespace>
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

Set local placeholder values for object storage:

```dotenv
COMPANION_AWS_KEY=<local-placeholder-key>
COMPANION_AWS_SECRET=<local-placeholder-secret>
COMPANION_AWS_BUCKET=<local-placeholder-bucket>
COMPANION_AWS_REGION=<aws-region>
```

> [!WARNING]
> Never place production AWS credentials or keys in local setup files.

---

## 3. Configure the Frontend

Create the `.env` file for the React application:

```bash
cd ~/mlmodelscope2/mlmodelscope
cp .env.example .env
```

Configure the local backend endpoints:

```dotenv
REACT_APP_API_URL=http://localhost:8005
REACT_APP_EXPLANATION_API_URL=http://127.0.0.1:8090
REACT_APP_COMPANION_URL=http://localhost
REACT_APP_IS_LOCAL=true
```

Install frontend dependencies:

```bash
cd ~/mlmodelscope2/mlmodelscope
npm install
```

---

## 4. Configure the Explanation API

Set up the environment file for the AI explanation service:

```bash
cd ~/mlmodelscope2/mlmodelscope/explanation-api
cp .env.example .env
```

Configure a supported explanation provider in `.env` (for example, using OpenAI):

```dotenv
EXPLANATION_PROVIDER=openai
EXPLANATION_MODEL=gpt-4.1-mini
OPENAI_API_KEY=<local-openai-api-key>

CORS_ORIGIN=http://localhost:3000
HOST=127.0.0.1
PORT=8090
MAX_IMAGE_BYTES=10485760
MAX_IMAGE_ATTACHMENTS=4
```

> [!NOTE]
> Keep provider API keys strictly server-side—never prefix them with `REACT_APP_`. You can also configure Gemini, vLLM, or other OpenAI-compatible endpoints by adjusting the provider and base URL settings documented in `explanation-api/README.md`.

Install dependencies and run test verification using the pinned Node runtime:

```bash
cd ~/mlmodelscope2/mlmodelscope/explanation-api
nvm install
nvm use
npm install
npm test
```

---

## 5. Start the API Stack

Start the backend infrastructure (PostgreSQL, RabbitMQ, Jaeger tracing, and upload helper) using Docker Compose:

```bash
cd ~/mlmodelscope2/mlmodelscope-api
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d --build
```

### Useful Local Endpoints
* **API Service**: [http://localhost:8005](http://localhost:8005)
* **Frontend Web App**: [http://localhost:3000](http://localhost:3000)
* **Explanation API**: [http://127.0.0.1:8090](http://127.0.0.1:8090)
* **RabbitMQ Management**: [http://localhost:15672](http://localhost:15672)
* **Jaeger UI**: [http://localhost:16686](http://localhost:16686)

Check container statuses and follow logs:

```bash
docker compose ps
docker compose logs -f api
```

---

## 6. Optional: Run `python_api` Directly

If you prefer to debug the Python API service directly on your host machine:

```bash
cd ~/mlmodelscope2/mlmodelscope-api/python_api
conda create -n mlms-api python=3.8 -y
conda activate mlms-api
pip install -r requirements.txt
```

Export your local database and message queue connection variables:

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

Start the FastAPI server:

```bash
fastapi run api.py --reload --port 8005
```

---

## 7. Build the Local PyTorch Agent

To run models locally without cloning from remote GitHub during build, create a modified Dockerfile:

```bash
cd ~/mlmodelscope2
cp py-mlmodelscope/dockerfiles/pytorch/Dockerfile.cpu_pytorch2.0.1 /tmp/Dockerfile.cpu_pytorch-local
```

Edit `/tmp/Dockerfile.cpu_pytorch-local` to copy your local checkout instead of running `git clone`:

```dockerfile
# Replace the remote clone line:
# RUN git clone https://github.com/xlab-ub/py-mlmodelscope.git /py-mlmodelscope

# With a local copy:
COPY py-mlmodelscope /py-mlmodelscope
```

Build the local agent container image:

```bash
cd ~/mlmodelscope2
docker build \
  -t pytorch-agent:local \
  -f /tmp/Dockerfile.cpu_pytorch-local \
  .
```

---

## 8. Run the Local PyTorch Agent

Start the PyTorch agent container after RabbitMQ is healthy:

```bash
cd ~/mlmodelscope2/mlmodelscope-api

docker rm -f pytorch-agent 2>/dev/null || true

docker run -d \
  --name pytorch-agent \
  --network host \
  --shm-size=1g \
  --ulimit memlock=-1 \
  --ulimit stack=67108864 \
  --env-file .env \
  pytorch-agent:local
```

Inspect the container logs to ensure it registers and begins consuming:

```bash
docker logs -f pytorch-agent
```

The agent should subscribe to the queue: `agent-pytorch-amd64`.

---

## 9. Start the Explanation API & Frontend

Launch the explanation API service in its own terminal:

```bash
cd ~/mlmodelscope2/mlmodelscope/explanation-api
nvm use
npm start
```

In a separate terminal, launch the React frontend:

```bash
cd ~/mlmodelscope2/mlmodelscope
nvm use
npm run start
```

Access the web application at [http://localhost:3000](http://localhost:3000).

---

## 10. Test an Image Classification Model

1. Open the frontend at [http://localhost:3000](http://localhost:3000).
2. Choose a PyTorch torchvision **ResNet** model (e.g., `ResNet-50`).
3. Select or upload a test image.
4. Optionally enable **Explain this prediction** to invoke the XAI service.
5. Click **Run model and see results**.

> [!NOTE]
> For XAI v1, explanations are supported for torchvision `ResNet-18`, `ResNet-34`, `ResNet-50`, `ResNet-101`, and `ResNet-152` with a batch size of one.

---

## Troubleshooting

### The frontend shows no models
Confirm that the API container is running and able to query PostgreSQL:
```bash
cd ~/mlmodelscope2/mlmodelscope-api
docker compose ps
docker compose logs -f api
```

### The trial stays pending
Verify that RabbitMQ is healthy and the PyTorch agent is actively listening on the queue:
```bash
docker compose logs -f mq
docker logs -f pytorch-agent
```

### The PyTorch agent cannot reach RabbitMQ
* On Linux, `--network host` allows direct connection via `localhost`.
* On Docker Desktop (macOS/Windows), change `MQ_HOST` in `.env` to `host.docker.internal` so the container can resolve host network services.

### Port conflicts
Ensure the following local ports are free before launching:

* **Frontend**: `3000`
* **Explanation API**: `8090`
* **API**: `8005`
* **PostgreSQL**: `15432`
* **RabbitMQ**: `5672`
* **RabbitMQ Management**: `15672`
* **Jaeger UI**: `16686`

---

## Privacy & Security Best Practices

* Never commit `.env` or `.env.companion` files to version control.
* Keep LLM / explanation-provider API keys restricted to `mlmodelscope/explanation-api/.env`.
* Avoid hard-coding local credentials directly in `python_api/db.py` or `python_api/mq.py`.
* Always use placeholder tokens in documentation and example configs.
