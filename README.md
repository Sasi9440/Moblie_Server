# MobileLLMServer

A local LLM inference server built on top of [llama.cpp](https://github.com/ggml-org/llama.cpp) — designed to run large language models efficiently on local hardware, including mobile-class and edge devices. It exposes an OpenAI-compatible REST API, making it easy to integrate with any client that supports the OpenAI API format.

---

## How It Works

```
Client (mobile app / browser / API tool)
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
3. Any client — mobile app, browser, or tool — can send requests to the server just like it would to the OpenAI API

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
