Local LLM Stack with Docker Compose (NVIDIA GPU)
Self‑host Ollama and complementary LLM tools locally on a single NVIDIA GPU workstation using Docker Compose. This repository provides a modular setup that’s easy to start, stop, update, and extend.
✨ Features

Ollama for local model management & OpenAI‑compatible API
Open WebUI (formerly Ollama WebUI) for a clean, chat‑centric UI
(Optional) text-generation-webui for advanced power‑users
(Optional) API gateway (e.g., oai-compat-proxy) to normalize endpoints
NVIDIA GPU acceleration using nvidia-container-toolkit
Clean volumes, health checks, and network isolation
Prometheus + Grafana ready metrics (optional)
One‑command start via docker compose up -d


Works on Linux and Windows (using WSL2). macOS works for CPU-only runs, but this repo is geared for NVIDIA GPU acceleration.
