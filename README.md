# local-ai

Local LLM inference, vision processing, and Retrieval-Augmented Generation (RAG) stack running on `lord-of-the-pings` (Intel Core i5-9500, 16GB DDR4, Ubuntu Linux). No data leaves the machine.

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

## **Hardware & Performance Benchmarks**

Tested on **Ubuntu Linux**, **Intel Core i5-9500 (6 Cores / 6 Threads @ 3.00GHz)**, and **16GB DDR4 RAM**. Inference is executed locally via CPU-bound Ollama execution.

### **1. Model Performance & Inference Speed**
| Model | Parameters / Quantization | Prompt Eval Rate | Generation Speed (Tokens/Sec) | Test Payload / Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Qwen 2.5** | 7B (Q4_K_M) | 5.07 t/s | **5.04 t/s** | 338 tokens generated in 1m 7s (Deep reasoning & code) |
| **Qwen 2.5** | 3B | 71.80 t/s | **11.83 t/s** | 204 tokens generated in 26.5s (Low-latency queries) |

### **2. System RAM Footprint & Resource Impact**
| System State | Used RAM | Available RAM | Model RAM Delta |
| :--- | :--- | :--- | :--- |
| **Baseline (Idle)** | 6.4 GiB | 8.1 GiB | — |
| **Active (3B Model Resident)** | 8.2 GiB | 6.4 GiB | **~1.8 GiB** |
| **Active (7B Model Resident)** | 10.0 GiB | 4.2 GiB | **~3.6 GiB** |

> **Engineering Takeaway:** On an LGA1151 CPU-bound platform, scaling down from the 7B to the 3B model more than doubles generation speed (**11.83 t/s** vs **5.04 t/s**) and halves the memory residency footprint (**~1.8 GiB** vs **~3.6 GiB**). With the 3B model resident, ~6.4 GiB of system memory remains available (~4.2 GiB with the 7B), leaving headroom for the other container services (Immich, Paperless-ngx, AnythingLLM).

---

### **Hardware Constraints & Scaling Path**

* **Current Limitation (CPU-Bound):** The i5-9500 integrated graphics (UHD 630) cannot accelerate transformer workloads. Inference relies strictly on CPU vector instructions (AVX2) and system RAM bandwidth, limiting efficient local execution to small quantized models (3B–7B parameters).
* **Impact of a Discrete GPU (VRAM Offloading):** Integrating a discrete GPU would shift model layers entirely into high-bandwidth VRAM, eliminating CPU memory-bandwidth bottlenecks, accelerating generation speeds drastically, and enabling full-layer offload for larger models.
* **Future Upgrade Roadmap (Planned):** Integration of an **NVIDIA T1000 (4GB or 8GB Low Profile)**.
  - *Form Factor & Power Fit:* Ideal for the HP SFF chassis, since it requires no external PCIe power cables (drawing under 50W entirely from the slot) and fits low-profile backplates.

---

## Design Decisions

* **Hybrid Native + Container Split:** Running Ollama directly on bare-metal Ubuntu avoids container overhead and gives `llama.cpp` direct access to the i5-9500's CPU threads and memory bandwidth, while AnythingLLM stays isolated in Docker.
* **Bridge Networking with Host Gateway:** AnythingLLM runs on an isolated Docker bridge network and communicates with the host Ollama daemon via `host.docker.internal:11434` (`host-gateway`) with zero external API calls, keeping local documents private.
* **Tailscale-Only Access:** The AnythingLLM port is published on the host's Tailscale address, never on the LAN or the internet. Ollama listens on all interfaces so the container can reach it, and the host firewall allows port `11434` only from the tailnet and the AnythingLLM Docker subnet.
* **Persistent Data:** Workspaces and vectors live in the external Docker volume `anythingllm_storage`, so recreating the container does not lose data.
* **Memory Budgeting (16GB RAM):** Measured footprint (see benchmarks above) is ~3.6 GiB for the 7B model and ~1.8 GiB for the 3B model, which lets them run alongside Paperless-ngx, Nginx, and host OS services. Models are loaded on demand rather than kept resident.

---

## Quick Start

Run the following commands to verify your Ollama models and launch the stack:

```bash
ollama list
docker volume create anythingllm_storage
cp .env.example .env
docker compose up -d
```

In `compose.yaml`, replace the port binding with your own Tailscale address (`"<tailscale-ip>:3001:3001"`).

### Firewall requirement

Allow the AnythingLLM Docker subnet to reach Ollama, or chat requests will time out:

```bash
sudo ufw allow from <anythingllm-subnet> to any port 11434 proto tcp comment 'AnythingLLM to Ollama'
```

Find the subnet with `docker network inspect local-ai_default`.

Part of the [homelab](https://github.com/AurelSto/homelab) infrastructure.
