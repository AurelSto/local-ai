# Local AI Architecture Overview

System topology and data flow for the hybrid native/containerized AI stack running on `lord-of-the-pings` (Intel Core i5-9500, 16GB DDR4, Ubuntu Linux).

```mermaid
flowchart TD
    User([User / Browser]) -->|HTTP :3001| AnythingLLM

    subgraph Host ["lord-of-the-pings (Ubuntu Bare Metal)"]
        subgraph Docker ["Docker Bridge Network"]
            AnythingLLM["AnythingLLM Container<br/>(RAG Pipeline & Web UI)"]
            LanceDB[("LanceDB<br/>(Vector Storage)")]
            Storage[("Docker Volume<br/>anythingllm_storage")]

            AnythingLLM <--> LanceDB
            AnythingLLM <--> Storage
        end

        subgraph Native ["Native systemd Services"]
            Ollama["Ollama Daemon<br/>(CPU / AVX2 Inference)<br/>:11434"]

            subgraph Models ["Local Model Registry"]
                LLM["qwen2.5:3b / 7b<br/>(Text & Code)"]
                VLM["qwen2.5vl:3b<br/>(Vision / Multimodal)"]
                Embed["nomic-embed-text<br/>(Embeddings)"]
            end

            Ollama --- LLM
            Ollama --- VLM
            Ollama --- Embed
        end

        AnythingLLM <-->|"host.docker.internal:11434<br/>(host-gateway)"| Ollama
    end
