# AIAssistant-on-UM790Pro-Accerated-by-ROCm
For AMD Pervasive Contest, using ROCm to support and accelerate LLMs running on UM790Pro AIPC, also enables local RAG.

## Contents

- [AIAssistant-on-UM790Pro-Accerated-by-ROCm](#aiassistant-on-um790pro-accerated-by-rocm)
  - [Contents](#contents)
  - [Task Brief](#task-brief)
    - [Choice of Acceleration Hardware](#choice-of-acceleration-hardware)
      - [GPU](#gpu)
  - [NPU](#npu)
  - [Build LLM Inference Server](#build-llm-inference-server)
    - [Add gfx1103 support for LM-Studio with ROCm](#add-gfx1103-support-for-lm-studio-with-rocm)
      - [Build rocblas for gfx1103](#build-rocblas-for-gfx1103)
      - [Install LM-studio](#install-lm-studio)
      - [Test(llama3.1-8B)](#testllama31-8b)
  - [Frontend](#frontend)
    - [Local RAG](#local-rag)
      - [Setup Up AnythingLLM](#setup-up-anythingllm)
      - [test with Q\&A with FlashAttention 1,2,3 paper pdf](#test-with-qa-with-flashattention-123-paper-pdf)
    - [Web browsing RAG](#web-browsing-rag)
    - [GraphRAG](#graphrag)
      - [Update Settings to support local LLMs](#update-settings-to-support-local-llms)

## Task Brief
The task is to support and accelerate LLMs running on UM790Pro AIPC, also enables local RAG, which helps users to work efficiently and privately.

### Choice of Acceleration Hardware
**Why I choose GPU instead of NPU for acceleration?** 

This UM790Pro AIPC contains accelerators both GPU and NPU. Here I want to explain why at present it is a better choice to use GPU for acceleration. I will explain in terms of Computing Capabilities and Usability(community supports).

**Note: As known to all, LLM inference tasks are more memory bound than CNN type models. The following Computing Capabilities may not directly show the performance of a certain type of Accelerator.**

#### GPU
***Computing Capabilities***

Running at 2.9GHz, the processor can deliver [4.51 TFLOPS](https://www.cpu-monkey.com/en/igpu-amd_radeon_780m) of FP32 performance, 9.03 TFLOPS of FP16 performance, and potentially **18.04 TOPS** of INT8 performance(4 times FP32). The INT8 performance figures are supported by [AMD's documentation](https://www.amd.com/zh-cn/newsroom/press-releases/2024-4-16-amd-expands-commercial-ai-pc-portfolio-to-deliver-.html), which indicates that the 8945HS model achieves 39 TOPS. By subtracting the 16 TOPS attributed to XDNA (the only significant difference from the 7945HS model) and adjusting for other CPU differences, we can infer the INT8 performance for this processor.

***Usability***

Granted with more support with OpenCL and **ROCm
**(not fully supported officially by AMD, we' ll solve this later). 
## NPU 
***Computing Capabilities***

NPU in UM790Pro(7940HS) is capable of Up to **10 TOPS** in [AMD's documentation](https://www.amd.com/en/products/processors/laptop/ryzen/7000-series/amd-ryzen-9-7940hs.html). 

![img](github_images/NPU-Performance.jpg)

Test results shows that longer token length shows lower NPU utilization rate and more volatile. And command center shows 70% of NPU utilization.

![img](github_images/NPU-Utilization.jpg)
![img](github_images/NPU-Utilization-long.jpg)

In the above pictures, the first tests from 4 to 256 tokens, the second tests 512 to 4096 tokens. Both increases with a factor of 2.

***Usability***

However, the NPU is not as widely supported as the GPU in terms of **Ops** and **Quantization** by communities. After I tried [examples](https://github.com/amd/RyzenAI-SW) of NPU, I found it **less flexible** and need **more time for adaptation** for each LLM model.

## Build LLM Inference Server
### Add gfx1103 support for LM-Studio with ROCm
#### Build rocblas for gfx1103
I refer to this page to help me rebuild rocblas for gfx1103 on Windows: [ROCm-Developer-Tools/rocBLAS](https://www.bilibili.com/read/cv34438089/?jump_opus=1). Or you can download it from [link](https://github.com/likelovewant/ROCmLibs-for-gfx1103-AMD780M-APU).
And you can get the following files:

![img](github_images/rocblas-files.jpg)

#### Install LM-studio
VERY IMPORTANT: You need to install windows version and get ROCm extension instead of ROCm preview from lmstudio.ai.

```bash
LM-Studio-0.2.28-Setup 
```
After this setup, run the folling command in Powershell to add support fot ROCm.

**please first delete everything in your %USERPROFILE%\\.cache\lm-studio\extensions\backends (Windows).**

```
Invoke-Expression ([System.Text.Encoding]::UTF8.GetString((Invoke-WebRequest -Uri https://files.lmstudio.ai/windows/extension-pack-install-scripts/win-rocm-0.2.27-ext-install.ps1 -UseBasicParsing).Content))
```
Alter backend-manifest.json in C:\Users\15824\\.cache\lm-studio\extensions\backends\win-llama-rocm-lm\ and add gfx1103 in "targets". The result should look like:
```json
{
  "name": "ROCm llama.cpp",
  "version": "1.1.0",
  "engine": "llama.cpp",
  "target_libraries": [
    {
      "name": "llm_engine_rocm.node",
      "type": "llm_engine",
      "version": "0.1.1"
    },
    {
      "name": "liblmstudio_bindings_rocm.node",
      "type": "liblmstudio",
      "version": "0.2.25"
    }
  ],
  "platform": "win",
  "cpu": {
    "architecture": "x86_64",
    "instruction_set_extensions": [
      "AVX2"
    ]
  },
  "gpu": {
    "make": "AMD",
    "framework": "ROCm",
    "targets": [
      "gfx1030",
      "gfx1100",
      "gfx1101",
      "gfx1102",
      "gfx1103"
    ]
  },
  "supported_model_formats": [
    "gguf"
  ],
  "manifest_version": "2",
  "vendor_lib_package_name": "win-llama-rocm-vendor"
}
```
Besides, replace old rocblas.dll(C:\Users\15824\.cache\lm-studio\extensions\backends\vendor\win-llama-rocm-vendor) and library fold(C:\Users\15824\.cache\lm-studio\extensions\backends\vendor\win-llama-rocm-vendor\rocblas) with the new one you build.


Then, download ollama ollama-windows-amd64.zip from [link](https://github.com/likelovewant/ollama-for-amd/releases). And replace the old llama.dll(C:\Users\15824\\.cache\lm-studio\extensions\backends\win-llama-rocm-lm) with Ollama's llama.dll(in ollama-windows-amd64\ollama_runners\rocm_v5.7).

#### Test(llama3.1-8B)
Test with newest model: Llama3.1-8B with 8-bit precision.
When you load model, you can the same as the picture below which indicates that the model is loaded successfully with llama.cpp support on ROCm backend.
![img](github_images/load-model.jpg)

Ask a few questions:
![img](github_images/ask-questions-1.jpg)
![img](github_images/ask-questions-2.jpg)

Token Generation Speed compared with NPU:
Note that it is not a fair comparison.
Llama3.1 8B 8bit
![img](github_images/GPU-token-generation-speed.jpg)

Llama2 7B chat 4bit
![img](github_images/GPU-token-generation-speed-llama2-4bit.jpg)

| Model | Llama2-FlashAttn-AWQ-4bit | Llama3.1-8B.Q8_0.gguf | llama-2-7B-chat.Q4_0.gguf |
| --- | --- | --- | --- |
| DEVICE | AIE(NPU) | GPU(780M) | GPU(780M) |
| Token/s | 3.8 | 8.22 | 14.50 |


## Frontend

### Local RAG
#### Setup Up AnythingLLM
First, Downliad anythingllm desktop.
Second, Set the LLM preference in settings of AntyhingLLM to LMStudio, and let the app auto detect the ip of the server, as well as the model.

![img](github_images/anythingllm-settings-llm.jpg)

Third, you can pick the embedding models, either the default from anythingllm or the one from LMStudio.

Finally, in agent skills. Open Web Search, and select Google Search Engine(because it is free). Enter your Search engine ID and Programmatic Access APIKey.

![img](github_images/anythingllm-agent-skills-websearch.jpg)

#### test with Q&A with FlashAttention 1,2,3 paper pdf
Local RAG with FlashAttention 1,2,3 paper pdf.
First, you should add the pdf file to the vector database. Something like this:

![img](github_images/anythingllm-add-pdf.jpg)

Then, ask questions concerning the pdf file. You can see the results below, which is quite accurate:

![img](github_images/anythingllm-ask-questions-pdf.jpg)
Here is the original Figure1 in pdf.
![img](github_images/anythingllm-ask-questions-pdf-2.jpg)

### Web browsing RAG
After setting up Google Search Engine, you can ask questions outside the training knowledge. 
Select Web Search in Agent Skills, 
![img](github_images/anythingllm-agent-skills-websearch-2.jpg)

Then, you can search the web
and ask questions. You can see the results below:

![img](github_images/anythingllm-agent-skills-websearch-3.jpg)
The output is different from local vector base RAG, and it will be much slower, for it precesses the web search in real time.

### GraphRAG
#### Update Settings to support local LLMs
