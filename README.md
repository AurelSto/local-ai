# local-ai

Local LLM inference, vision processing, and Retrieval-Augmented Generation (RAG) stack running on `lord-of-the-pings` (Intel Core i5-9500, 16GB DDR4, Ubuntu Linux).

## Architecture

For a complete visual system topology and data flow map, see the [Architecture Overview](ARCHITECTURE.md).

| Layer | Component | Deployment | Role |
| :--- | :--- | :--- | :--- |
| **Inference Engine** | Ollama | Native (`systemd`) | Direct CPU/AVX2 execution of quantized GGUF models on port `11434` |
| **Language Models** | `qwen2.5:3b`, `qwen2.5:7b` | Local Ollama registry | `3b` for low-latency queries; `7b` for deeper reasoning and code tasks |
| **Vision Model** | `qwen2.5vl:3b` | Local Ollama registry | Multimodal image and document analysis |
| **Embedding Model** | `nomic-embed-text` | Local Ollama registry | Text vectorization for AnythingLLM RAG workspaces |
| **UI & RAG Pipeline** | AnythingLLM | Docker (`mintplexlabs/anythingllm`) | Document ingestion, local LanceDB vector store, and chat workspace on port `3001` |

---

## Design Decisions

* **Hybrid Native + Container Split:** Running Ollama directly on bare-metal Ubuntu avoids container overhead and gives `llama.cpp` direct access to the i5-9500's CPU threads and memory bandwidth, while AnythingLLM stays isolated in Docker.
* **Bridge Networking with Host Gateway:** AnythingLLM runs on an isolated Docker bridge network and communicates with the host Ollama daemon via `host.docker.internal:11434` (`host-gateway`) with zero external API calls—ensuring 100% data privacy for local documents.
* **Memory Budgeting (16GB RAM):** Carefully budgeted to allow `qwen2.5:7b` (~4.7GB RAM) and supporting models to run comfortably alongside Paperless-ngx, Nginx, and host OS services.

---

## Quick Start

Run the following commands to verify your Ollama models and launch the stack:

```bash
ollama list
cp .env.example .env
docker compose up -d
```

Part of the [homelab](https://github.com/AurelSto/homelab) infrastructure. 
