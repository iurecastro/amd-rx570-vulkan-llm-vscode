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
- [Step 2: Set up llama.cpp](#step-2-set-up-llamacpp)
- [Step 3: Run the server with GPU offload](#step-3-run-the-server-with-gpu-offload)
- [Step 4: Integrate with VS Code](#step-4-integrate-with-vs-code)
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

1. Download the latest **Vulkan SDK** from [LunarG](https://vulkan.lunarg.com/).
2. Install with default components.
3. Verify your GPU is detected using **Vulkan Hardware Capability Viewer** or the CLI.

---

## Step 2: Set up `llama.cpp`

1. Create a folder, for example:
   ```powershell
   C:\llama
   ```

2. Download the precompiled Windows binaries for `llama.cpp` with **Vulkan support**.
3. Place `llama-server.exe` inside `C:\llama`.
4. Test device discovery:
   ```powershell
   cd C:\llama
   .\llama-server.exe --list-devices
   ```

   Expected output:
   ```
   Available devices:
     Vulkan0: Radeon RX 570 Series (8192 MiB, ...)
   ```

---

## Step 3: Run the server with GPU offload

Start the server manually:

```powershell
cd C:\llama
.\llama-server.exe -m "C:\path\to\models\qwen2.5-coder-7b.gguf" -ngl 99 --port 1234
```

- `-ngl 99` → offloads all layers to VRAM.
- `--port 1234` → sets the local OpenAI-compatible endpoint.

### Optional: one-click startup script

Create `start-llama.bat` inside `C:\llama`:

```cmd
@echo off
cd /d C:\llama
llama-server.exe -m "C:\path\to\models\qwen2.5-coder-7b.gguf" -ngl 99 --port 1234
pause
```

Double-click it whenever you want to start the server.

---

## Step 4: Integrate with VS Code

VS Code supports local custom endpoints natively.

1. Open the language model configuration file:
   ```
   %APPDATA%\Code\User\chatLanguageModels.json
   ```

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
| Server not responding | Run `.\llama-server.exe --list-devices` and confirm Vulkan0 is detected. |
| Slow generation | Close other GPU-heavy apps. Reduce context size. Ensure `-ngl 99` is used. |
| VS Code doesn't show the model | Verify the JSON syntax. Restart VS Code after saving. |

---

## Repository Structure

```
.
├── README.md
├── LICENSE
├── start-llama.bat
└── images/
    ├── task-manager.png
    └── vscode-chat.png
```

---

## License

This project is licensed under the **MIT License**. See the [LICENSE](LICENSE) file for details.

---

⭐ If this guide helped you, consider starring the repository!
