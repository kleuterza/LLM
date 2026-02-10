Below is a complete, production‑ready **README.md** you can drop into your repo. It’s optimized for a **local NVIDIA workstation** hosting **Ollama** and a small set of **LLM components** via **Docker Compose**. I’ve included clear setup steps, environment variables, security notes, health checks, and optional add‑ons (Open WebUI, text-generation-webui, API gateway, and metrics). Adjust names/ports to match your repo.

***

# Local LLM Stack with Docker Compose (NVIDIA GPU)

Self‑host **Ollama** and complementary LLM tools locally on a **single NVIDIA GPU workstation** using **Docker Compose**. This repository provides a modular setup that’s easy to start, stop, update, and extend.

## ✨ Features

*   **Ollama** for local model management & OpenAI‑compatible API
*   **Open WebUI** (formerly Ollama WebUI) for a clean, chat‑centric UI
*   **(Optional)** `text-generation-webui` for advanced power‑users
*   **(Optional)** API gateway (e.g., `oai-compat-proxy`) to normalize endpoints
*   **NVIDIA GPU** acceleration using `nvidia-container-toolkit`
*   Clean **volumes**, **health checks**, and **network isolation**
*   **Prometheus + Grafana** ready metrics (optional)
*   One‑command start via `docker compose up -d`

> Works on Linux and Windows (using WSL2). macOS works for CPU-only runs, but this repo is geared for **NVIDIA GPU** acceleration.

***

## 🧭 Repository Structure

    .
    ├─ compose/
    │  ├─ docker-compose.yml            # Core stack (Ollama + Open WebUI)
    │  ├─ docker-compose.power.yml      # Adds text-generation-webui, gateway, metrics
    │  ├─ profiles/                     # Optional service profiles
    │  │  ├─ monitoring.yml             # Prometheus + Grafana
    │  │  └─ gateway.yml                # OpenAI-compatible gateway
    ├─ env/
    │  ├─ .env.example                  # Environment variable template
    │  └─ secrets/                      # Place secrets here (not committed)
    ├─ grafana/
    │  └─ provisioning/                 # Example dashboards/datasources
    ├─ prometheus/
    │  └─ prometheus.yml                # Example scrape config
    ├─ scripts/
    │  ├─ pull_models.sh                # Pre-pull common Ollama models
    │  └─ refresh.sh                    # Update images safely
    └─ README.md                        # This file

***

## 🧩 What’s Included

### Core

*   **Ollama** – local model runner & manager, exposes:
    *   REST: `http://localhost:11434`
    *   OpenAI‑compatible endpoint: `http://localhost:11434/v1`
*   **Open WebUI** – browser UI for chat, model switching, and file attachments
    *   UI: `http://localhost:3000`

### Optional Add‑Ons

*   **text-generation-webui** – power user web UI with extensions (`http://localhost:7860`)
*   **OpenAI-compatible gateway** – normalize varying backends under a single endpoint (`http://localhost:8080/v1`)
*   **Prometheus + Grafana** – metrics & dashboards (`http://localhost:9090`, `http://localhost:3001`)

***

## 📦 Prerequisites

1.  **NVIDIA GPU** with recent drivers (CUDA‑capable).
2.  **Docker** and **Docker Compose** installed.
3.  **NVIDIA Container Toolkit** installed and configured.

### Quick Checks

```bash
# Docker + Compose
docker --version
docker compose version

# NVIDIA toolkit
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.3.2-base-ubuntu22.04 nvidia-smi
```

> If the CUDA container prints your GPU details, you’re good to go.

***

## ⚙️ Environment

Copy the template and adjust as needed:

```bash
cp env/.env.example .env
```

**Key variables** (example):

```env
# Ollama
OLLAMA_HOST=0.0.0.0
OLLAMA_PORT=11434
OLLAMA_MODELS=/data/ollama

# Open WebUI
WEBUI_PORT=3000

# text-generation-webui (optional)
TGI_PORT=7860

# Gateway (optional)
GATEWAY_PORT=8080

# Monitoring (optional)
PROM_PORT=9090
GRAFANA_PORT=3001

# Auth (optional)
WEBUI_ADMIN_USER=admin
WEBUI_ADMIN_PASS=change-me
```

> Place secrets (tokens, passwords) in `env/secrets/` and **do not commit** them.

***

## 🧪 Quick Start

1.  **Start the core stack** (Ollama + Open WebUI):

```bash
cd compose
docker compose --env-file ../.env -f docker-compose.yml up -d
```

2.  **Pull a model** (examples):

```bash
docker exec -it ollama ollama pull llama3.1:8b
docker exec -it ollama ollama pull qwen2.5:7b
```

3.  **Open the UI**: `http://localhost:${WEBUI_PORT:-3000}`

4.  **Test the API**:

```bash
curl http://localhost:${OLLAMA_PORT:-11434}/api/generate -d '{
  "model":"llama3.1:8b",
  "prompt":"Hello! Summarize your capabilities in one sentence."
}'
```

***

## 🧰 Core Docker Compose (Ollama + Open WebUI)

> File: `compose/docker-compose.yml`

```yaml
name: local-llm

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "${OLLAMA_PORT:-11434}:11434"
    environment:
      - OLLAMA_HOST=${OLLAMA_HOST:-0.0.0.0}
    volumes:
      - ollama_models:${OLLAMA_MODELS:-/data/ollama}
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: ["gpu"]
    runtime: nvidia
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:11434/ || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 20

  open-webui:
    image: ghcr.io/open-webui/open-webui:latest
    container_name: open-webui
    restart: unless-stopped
    depends_on:
      ollama:
        condition: service_healthy
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_AUTH=True
      - WEBUI_ADMIN_USER=${WEBUI_ADMIN_USER:-admin}
      - WEBUI_ADMIN_PASS=${WEBUI_ADMIN_PASS:-change-me}
    ports:
      - "${WEBUI_PORT:-3000}:8080"
    volumes:
      - webui_data:/app/backend/data

volumes:
  ollama_models:
  webui_data:
```

> **GPU note:** `runtime: nvidia` is respected if the NVIDIA toolkit is installed. Some Docker versions use the `--gpus all` equivalent via `deploy.resources.reservations.devices`.

***

## 🔌 Optional Services (Power Profile)

> File: `compose/docker-compose.power.yml` (extends core)

```yaml
services:
  textgen:
    image: ghcr.io/oobabooga/text-generation-webui:latest
    container_name: textgen-webui
    restart: unless-stopped
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: ["gpu"]
    environment:
      - CLI_ARGS=--listen --listen-port 7860 --api
    ports:
      - "${TGI_PORT:-7860}:7860"
    volumes:
      - textgen_data:/data
    healthcheck:
      test: ["CMD", "bash", "-lc", "curl -sf http://localhost:7860/ || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 20

  gateway:
    image: ghcr.io/vltgroup/oai-compat-proxy:latest
    container_name: oai-gateway
    restart: unless-stopped
    depends_on:
      ollama:
        condition: service_healthy
    environment:
      - UPSTREAMS=ollama:http://ollama:11434
      - DEFAULT_UPSTREAM=ollama
    ports:
      - "${GATEWAY_PORT:-8080}:8080"

  prometheus:
    image: prom/prometheus:latest
    container_name: prom
    restart: unless-stopped
    volumes:
      - ../prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "${PROM_PORT:-9090}:9090"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    depends_on:
      - prometheus
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=change-me
    volumes:
      - grafana_data:/var/lib/grafana
      - ../grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "${GRAFANA_PORT:-3001}:3000"

volumes:
  textgen_data:
  grafana_data:
```

**Start with optional services:**

```bash
docker compose --env-file ../.env \
  -f docker-compose.yml \
  -f docker-compose.power.yml up -d
```

***

## 🧪 Testing the OpenAI-Compatible Path

Use the **gateway** (if enabled) or Ollama’s `/v1` directly:

```bash
# Using gateway
curl http://localhost:${GATEWAY_PORT:-8080}/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.1:8b",
    "messages": [{"role":"user","content":"Explain RAG in 2 sentences."}]
  }'

# Direct to Ollama’s OpenAI-compatible endpoint
curl http://localhost:${OLLAMA_PORT:-11434}/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.1:8b",
    "messages": [{"role":"user","content":"Explain RAG in 2 sentences."}]
  }'
```

**Python example (OpenAI SDK pointing to Ollama):**

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

resp = client.chat.completions.create(
    model="llama3.1:8b",
    messages=[{"role": "user", "content": "Give me a one-line haiku about GPUs."}]
)

print(resp.choices[0].message.content)
```

***

## 📁 Model Management

**List models:**

```bash
docker exec -it ollama ollama list
```

**Pull a model:**

```bash
docker exec -it ollama ollama pull mistral:7b
```

**Delete a model:**

```bash
docker exec -it ollama ollama rm mistral:7b
```

**Create a local model (Modelfile):**

```bash
docker exec -it ollama bash -lc 'cat > /tmp/Modelfile <<EOF
FROM llama3.1:8b
PARAMETER temperature 0.2
SYSTEM You are a concise assistant.
EOF
ollama create llama3.1- concise -f /tmp/Modelfile'
```

***

## 🔐 Security Considerations

*   **Do not expose** these services to the public internet without network controls.
*   Use a **reverse proxy** (Traefik, Caddy, or Nginx) with **TLS** and **basic auth/OIDC** if remote access is required.
*   Store secrets under `env/secrets/` and reference via `--env-file` or Docker secrets.
*   Limit GPU & CPU **resource usage** per service where needed.
*   Prefer a **private LAN** or **VPN** for any remote access.

***

## 🧭 Networking & Ports

| Service               | Port (host) | Internal |
| --------------------- | ----------- | -------- |
| Ollama API            | 11434       | 11434    |
| Open WebUI            | 3000        | 8080     |
| text-generation-webui | 7860        | 7860     |
| OpenAI Gateway        | 8080        | 8080     |
| Prometheus            | 9090        | 9090     |
| Grafana               | 3001        | 3000     |

> Adjust via `.env`.

***

## 🛠️ Maintenance

**Update images safely:**

```bash
cd compose
docker compose pull
docker compose down
docker compose up -d
```

**View logs:**

```bash
docker compose logs -f ollama
docker compose logs -f open-webui
```

**Health checks:**

```bash
curl -f http://localhost:${OLLAMA_PORT:-11434}/ || echo "Ollama not healthy"
```

***

## 🧯 Troubleshooting

*   **`failed to initialize NVML` or `no CUDA devices`**  
    Ensure NVIDIA drivers and `nvidia-container-toolkit` are installed. Test with the `nvidia/cuda` container (see prerequisites).

*   **High VRAM usage / OOM**  
    Use smaller quantized models (e.g., `Q4_K_M`), or choose 7B class models. Shut down unused services.

*   **Slow first response**  
    First tokenization & load is slower; subsequent prompts are faster due to caching.

*   **WSL2**  
    Ensure WSL2 is using NVIDIA CUDA host integration. Update WSL and drivers; enable GPU compute in WSL settings.

***

## 📈 (Optional) Monitoring

*   Enable `prometheus` and `grafana` via the power compose file and import dashboards from `grafana/provisioning`.
*   Consider exporters:
    *   **Node exporter** for host metrics
    *   **DCGM exporter** for GPU stats

***

## 🧩 Extending the Stack

*   Add **RAG** components (e.g., **Qdrant**, **Chroma**, **Weaviate**) and point your apps at `ollama` for generation.
*   Add a **reverse proxy** (Traefik) for TLS + auth:
    *   Map labels on services and expose via HTTPS
*   Add **LM Studio** (local inference UI) or other gateways as needed.

***

## 📜 License

Choose a license appropriate for your use case, e.g., **MIT** or **Apache-2.0**. Place it as `LICENSE` at the repo root.

***

## 🙌 Credits

*   **Ollama** for the local model runtime
*   **Open WebUI** for the UI
*   **Oobabooga text-generation-webui** for a powerful alternative UI
*   **Prometheus/Grafana** for observability

***

## ✅ Quick Commands Recap

```bash
# Start core
docker compose --env-file ../.env -f compose/docker-compose.yml up -d

# Start with power profile
docker compose --env-file .env \
  -f compose/docker-compose.yml \
  -f compose/docker-compose.power.yml up -d

# Pull models
docker exec -it ollama ollama pull llama3.1:8b

# Test API
curl http://localhost:11434/api/tags

# Stop
docker compose -f compose/docker-compose.yml down
```

***

If you’d like, I can tailor this README to your **exact repo layout** (filenames, profiles, and which optional services you want by default) or add a **Traefik** config with TLS and basic auth.

