https://github.com/user-attachments/assets/9c1c2ae8-ceb5-4f05-92dc-7a3e7a29a75e

# MobileLLMServer

A local LLM inference server built on top of [llama.cpp](https://github.com/ggml-org/llama.cpp) — designed to run large language models efficiently on local hardware, including mobile-class and edge devices. It exposes an OpenAI-compatible REST API, making it easy to integrate with any client that supports the OpenAI API format.

---

## How It Works

```
Client (mobile app / browser / API tool)
        │
        ▼
  Cloudflare Tunnel (public HTTPS URL)
        │
        ▼
  llama-server (HTTP REST API on port 8080)
        │
        ▼
  llama.cpp (C++ inference engine)
        │
        ▼
  GGUF Model (quantized LLM on CPU/GPU)
```

1. **llama.cpp** loads a GGUF-format model file into memory and runs inference using optimized C++ kernels
2. **llama-server** wraps the inference engine with an HTTP server exposing OpenAI-compatible endpoints (`/v1/chat/completions`, `/v1/completions`, `/embedding`, etc.)
3. **Cloudflare Tunnel** (`cloudflared`) creates a secure public HTTPS URL that forwards traffic to your local server — no port forwarding or static IP needed
4. Any client — mobile app, browser, or tool — can send requests to the public Cloudflare URL just like it would to the OpenAI API

---

## Prerequisites

- **CMake** >= 3.14
- **C++17 compiler** (MSVC, GCC, or Clang)
- A **GGUF model file** (download from [Hugging Face](https://huggingface.co/models?library=gguf&sort=trending))
- *(Optional)* CUDA toolkit for NVIDIA GPU acceleration

---

## Setup & Build

### 1. Clone the repository

```bash
git clone https://github.com/Sasi9440/Moblie_Server.git
cd Moblie_Server
```

### 2. Build (CPU only)

```bash
cd llama.cpp
cmake -B build
cmake --build build --config Release -j 8
```

### 3. Build with NVIDIA GPU (CUDA)

```bash
cd llama.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j 8
```

### 4. Build with Vulkan (cross-platform GPU)

```bash
cd llama.cpp
cmake -B build -DGGML_VULKAN=ON
cmake --build build --config Release -j 8
```

---

## Download a Model

Download any GGUF model from Hugging Face. Example using the CLI:

```bash
# Download and run directly (auto-downloads from Hugging Face)
./build/bin/llama-server -hf ggml-org/gemma-3-1b-it-GGUF
```

Or manually download a `.gguf` file and place it in the `models/` directory.

---

## Start the Server

```bash
# Basic server on port 8080
./build/bin/llama-server -m models/your-model.gguf --port 8080

# With GPU offloading (offload all layers to GPU)
./build/bin/llama-server -m models/your-model.gguf --port 8080 -ngl 99

# Support multiple concurrent users (4 parallel slots, 16K context)
./build/bin/llama-server -m models/your-model.gguf -c 16384 -np 4 --port 8080
```

Once running:
- Web UI → `http://localhost:8080`
- Chat completions → `http://localhost:8080/v1/chat/completions`
- Text completions → `http://localhost:8080/v1/completions`
- Embeddings → `http://localhost:8080/v1/embeddings`

---

## API Usage

### Chat Completion (OpenAI-compatible)

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "local",
    "messages": [
      { "role": "user", "content": "Hello! Who are you?" }
    ]
  }'
```

### Text Completion

```bash
curl http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Once upon a time",
    "n_predict": 100
  }'
```

---

## Public Deployment with Cloudflare Tunnel

Cloudflare Tunnel exposes your local `llama-server` to the internet securely without opening firewall ports or needing a static IP. It works by running a lightweight `cloudflared` daemon on your machine that connects outbound to Cloudflare's edge network.

### How it works

```
Internet Client
      │
      ▼
 Cloudflare Edge (your-tunnel.trycloudflare.com)
      │  (encrypted outbound tunnel)
      ▼
 cloudflared (running on your machine)
      │
      ▼
 llama-server (localhost:8080)
```

### Quick Start (no account needed)

The fastest way — generates a temporary public URL instantly:

```bash
# Install cloudflared (Windows)
winget install Cloudflare.cloudflared

# Install cloudflared (macOS)
brew install cloudflared

# Install cloudflared (Linux)
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
chmod +x cloudflared && sudo mv cloudflared /usr/local/bin
```

Start your llama-server first, then run the tunnel:

```bash
# Step 1 — start the LLM server
./build/bin/llama-server -m models/your-model.gguf --port 8080

# Step 2 — in a new terminal, expose it publicly
cloudflared tunnel --url http://localhost:8080
```

You'll see output like:

```
INF +--------------------------------------------------------------------------------------------+
INF |  Your quick Tunnel has been created! Visit it at (it may take some time to be reachable):  |
INF |  https://your-random-name.trycloudflare.com                                                |
INF +--------------------------------------------------------------------------------------------+
```

Your server is now publicly accessible at `https://your-random-name.trycloudflare.com`

### Persistent Tunnel (with Cloudflare account)

For a stable, named URL that survives restarts:

```bash
# 1. Login to Cloudflare
cloudflared tunnel login

# 2. Create a named tunnel
cloudflared tunnel create llm-server

# 3. Route your domain to the tunnel
cloudflared tunnel route dns llm-server llm.yourdomain.com

# 4. Start the tunnel
cloudflared tunnel run --url http://localhost:8080 llm-server
```

### Run as a Background Service

```bash
# Install as a system service (runs on boot)
sudo cloudflared service install
sudo systemctl start cloudflared   # Linux
```

On Windows, run in an elevated terminal:

```cmd
cloudflared service install
sc start cloudflared
```

### Use the Public URL with any OpenAI client

Once the tunnel is running, replace `localhost:8080` with your Cloudflare URL:

```bash
curl https://your-random-name.trycloudflare.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "local",
    "messages": [
      { "role": "user", "content": "Hello from the internet!" }
    ]
  }'
```

Or configure any OpenAI-compatible SDK:

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://your-random-name.trycloudflare.com/v1",
    api_key="none"  # no key needed for local server
)

response = client.chat.completions.create(
    model="local",
    messages=[{"role": "user", "content": "Hello!"}]
)
print(response.choices[0].message.content)
```

> **Note:** The quick tunnel URL changes every time you restart `cloudflared`. Use a persistent named tunnel with your own domain for a stable URL.

---

## CLI Usage (without server)

```bash
# Interactive chat
./build/bin/llama-cli -m models/your-model.gguf

# Single prompt
./build/bin/llama-cli -m models/your-model.gguf -p "What is the capital of France?" -n 100
```

---

## Supported Hardware Backends

| Backend | Platform |
|---|---|
| CPU (default) | All platforms |
| CUDA | NVIDIA GPUs |
| Vulkan | Cross-platform GPUs |
| Metal | Apple Silicon (macOS/iOS) |
| HIP | AMD GPUs |
| OpenCL | Adreno (Android/Snapdragon) |
| SYCL | Intel GPUs |

---

## Supported Model Families

LLaMA 1/2/3, Mistral, Mixtral, Gemma, Phi, Qwen, DeepSeek, Falcon, GPT-2, BERT, and [many more](llama.cpp/README.md).

---

## Project Structure

```
MobileLLMServer/
├── llama.cpp/          # Core inference engine (C++)
│   ├── src/            # llama library source
│   ├── tools/server/   # HTTP server (llama-server)
│   ├── tools/cli/      # CLI tool (llama-cli)
│   ├── ggml/           # Low-level tensor/math library
│   ├── common/         # Shared utilities
│   └── models/         # Vocabulary/model files
└── README.md
```

---

## License

This project uses [llama.cpp](https://github.com/ggml-org/llama.cpp) which is licensed under the [MIT License](llama.cpp/LICENSE).
