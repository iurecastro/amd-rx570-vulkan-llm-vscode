# Run Qwen 2.5 Coder 7B on AMD RX 570 with Vulkan and VS Code

![Platform](https://img.shields.io/badge/platform-Windows-0078D6?logo=windows)
![GPU](https://img.shields.io/badge/GPU-AMD%20RX%20570-ED1C24?logo=amd)
![Backend](https://img.shields.io/badge/backend-Vulkan-AC162C?logo=vulkan)
![License](https://img.shields.io/badge/license-MIT-green)

> A complete guide to running a local coding LLM entirely on an older AMD GPU using Vulkan and `llama.cpp`, then integrating it with VS Code's native chat.

---

## Table of Contents

- [Overview](#overview)
- [Why Vulkan?](#why-vulkan)
- [Prerequisites](#prerequisites)
- [Step 1: Install Vulkan SDK](#step-1-install-vulkan-sdk)
- [Step 2: Download llama.cpp (Vulkan build)](#step-2-download-llamacpp-vulkan-build)
- [Step 3: Download the Qwen2.5 Coder 7B model](#step-3-download-the-qwen25-coder-7b-model)
- [Step 4: Organize your folder structure](#step-4-organize-your-folder-structure)
- [Step 5: Run the server with GPU offload](#step-5-run-the-server-with-gpu-offload)
- [Step 6: Integrate with VS Code](#step-6-integrate-with-vs-code)
- [Performance](#performance)
- [Troubleshooting](#troubleshooting)
- [Repository Structure](#repository-structure)
- [License](#license)

---

## Overview

Running large language models locally usually depends on NVIDIA CUDA. However, older AMD GPUs like the **Radeon RX 570 (Polaris / gfx803)** do not have official ROCm support anymore.

This guide shows how to bypass that limitation using the **Vulkan backend** with `llama.cpp`. You will be able to:

- Offload all model layers to the GPU (`-ngl 99`)
- Run the model fully offline and privately
- Use it directly inside **VS Code's native chat**
- Achieve around **15–36 tokens/second** on an RX 570 8GB

---

## Why Vulkan?

- **No ROCm required** — works on older AMD cards.
- **Cross-platform** — Vulkan runs on Windows and Linux.
- **Full VRAM offload** — keeps the model on the GPU for fast inference.
- **OpenAI-compatible API** — `llama-server` exposes `/v1/chat/completions`.

---

## Prerequisites

| Component | Requirement |
|-----------|-------------|
| OS | Windows 10 / 11 |
| GPU | AMD Radeon RX 570 (8GB VRAM) |
| Runtime | Vulkan SDK (LunarG) |
| Inference Engine | `llama.cpp` (`llama-server.exe`) |
| Model | `qwen2.5-coder-7b-instruct` (GGUF) |
| Editor | Visual Studio Code (Native Chat) |

---

## Step 1: Install Vulkan SDK

The Vulkan SDK provides the runtime and drivers needed for GPU compute shaders.

1. Go to the official LunarG download page:  
   **[https://vulkan.lunarg.com/sdk/home#windows](https://vulkan.lunarg.com/sdk/home#windows)**
2. Download the **Windows x64** installer (e.g., `vulkansdk-windows-X64-1.4.363.0.exe`, ~289 MB).
3. Run the installer and keep the default components.
4. After installation, verify your GPU is detected by running the **Vulkan Hardware Capability Viewer** or using the CLI.

> **Note:** The Vulkan SDK is only required at runtime for `llama.cpp` to access the GPU. You do not need to compile anything.

---

## Step 2: Download llama.cpp (Vulkan build)

`llama.cpp` provides precompiled Windows binaries with Vulkan support.

1. Go to the official releases page:  
   **[https://github.com/ggml-org/llama.cpp/releases](https://github.com/ggml-org/llama.cpp/releases)**
2. Find the latest release that includes a **Windows x64 (Vulkan)** asset.  
   The file will be named something like:  
   `llama-bXXXX-bin-win-vulkan-x64.zip`
3. Download and extract the ZIP into `C:\llama`.  
   You should see `llama.exe` and its dependencies (DLLs) inside.

4. Test device discovery:
   ```powershell
   cd C:\llama
   .\llama.exe --list-devices
   ```

   Expected output:
   ```
   Available devices:
     Vulkan0: Radeon RX 570 Series (8192 MiB, ...)
   ```

> **Tip:** If you don't see `Vulkan0`, make sure the Vulkan SDK is installed and your GPU drivers are up to date.

---

## Step 3: Download the Qwen2.5 Coder 7B model

The model must be in **GGUF** format.

1. Go to the Hugging Face model page:  
   **[https://huggingface.co/Qwen/Qwen2.5-Coder-7B-Instruct-GGUF](https://huggingface.co/Qwen/Qwen2.5-Coder-7B-Instruct-GGUF)**
2. Choose a quantization that fits your VRAM. For an RX 570 8GB, **Q4_K_M** is a good balance.  
   Example file: `qwen2.5-coder-7b-instruct-q4_k_m.gguf` (~4.7 GB)
3. Download the `.gguf` file.

> **Alternative:** You can also use community GGUF repos like  
> [bartowski/Qwen2.5-Coder-7B-Instruct-GGUF](https://huggingface.co/bartowski/Qwen2.5-Coder-7B-Instruct-GGUF) or  
> [DhruvalLabs/Qwen2.5-Coder-7B-Instruct-GGUF](https://huggingface.co/DhruvalLabs/Qwen2.5-Coder-7B-Instruct-GGUF).

---

## Step 4: Organize your folder structure

Create the following layout inside `C:\llama`:

```
C:\llama\
├── llama-server.exe
├── llama.dll
├── ggml-vulkan.dll
├── ... (other llama.cpp DLLs)
├── models\
│   └── qwen2.5-coder-7b\
│       └── qwen2.5-coder-7b.gguf
├── logs\
├── start-llama.bat
└── stop-llama.bat
```

Move the downloaded `.gguf` file into:  
`C:\llama\models\qwen2.5-coder-7b\qwen2.5-coder-7b.gguf`

---

## Step 5: Run the server with GPU offload

You can start the server manually or via a batch script.

### Manual start

```powershell
cd C:\llama
.\llama.exe -m "C:\llama\models\qwen2.5-coder-7b\qwen2.5-coder-7b.gguf" -ngl 99 -c 32768 --port 1234
```

- `-ngl 99` → offloads all layers to VRAM.
- `-c 32768` → context size.
- `--port 1234` → sets the local OpenAI-compatible endpoint.

### Batch script: `start-llama.bat`

Create this file inside `C:\llama`:

```bat
cd /d C:\llama
llama-server.exe -m "C:\llama\models\qwen\qwen2.5-coder-7b\qwen2.5-coder-7b.gguf" -ngl 99 --port 1234
```

---

## Step 6: Integrate with VS Code

VS Code supports local custom endpoints natively via `chatLanguageModels.json`.

1. Open the language model configuration file:  
   `%APPDATA%\Code\User\chatLanguageModels.json`

2. Add or replace with:

   ```json
   [
     {
       "name": "Qwen 2.5 Coder",
       "vendor": "customendpoint",
       "apiType": "chat-completions",
       "models": [
         {
           "id": "qwen2.5-coder-7b",
           "name": "Qwen 2.5 Coder 7B",
           "url": "http://localhost:1234/v1/chat/completions",
           "toolCalling": false
         }
       ]
     }
   ]
   ```

3. Save the file and **restart VS Code**.
4. Open the native chat panel on the right side.
5. Select **Qwen 2.5 Coder 7B** from the model selector.
6. Start chatting and coding locally!

> **Note:** The `id` must match exactly what `llama-server` reports. If you used a different filename, update the `id` accordingly.

---

## Performance

| Metric | Value |
|--------|-------|
| VRAM Usage | ~6.7 GB / 8.0 GB |
| Speed | ~15–36 tokens/second |
| GPU Utilization | ~99% under load |
| Backend | Vulkan compute shaders |

> Results may vary depending on context length, prompt complexity, and driver version.

---

## Troubleshooting

| Problem | Possible Solution |
|---------|-------------------|
| `Language model unavailable` | Make sure `llama-server.exe` is running on port `1234`. Restart VS Code. |
| `The selected model ... is not available` | Check if the `id` in `chatLanguageModels.json` matches the model name served by `llama-server`. |
| Server not responding | Run `.\llama-server.exe --list-devices` and confirm `Vulkan0` is detected. |
| Slow generation | Close other GPU-heavy apps. Reduce context size. Ensure `-ngl 99` is used. |
| VS Code doesn't show the model | Verify the JSON syntax. Restart VS Code after saving. |
| `Vulkan0` not detected | Update your GPU drivers. Make sure the Vulkan SDK is installed. |

---

## Repository Structure

```
.
├── README.md
├── LICENSE
├── start-llama.bat
├── stop-llama.bat
└── images/
    ├── task-manager.png
    └── vscode-chat.png
```

---

## License

This project is licensed under the **MIT License**. See the [LICENSE](LICENSE) file for details.

---

⭐ If this guide helped you, consider starring the repository!
