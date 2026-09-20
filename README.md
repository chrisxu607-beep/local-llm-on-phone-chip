# Local LLM on Phone Chip

Running quantized large language models locally on a Qualcomm Snapdragon 8 Elite mobile SoC using ARM64 Linux, Vulkan, Mesa Turnip, and llama.cpp.

This project explores whether a modern phone-class processor can function as a practical local AI inference platform without relying on a discrete GPU or cloud inference.

## Overview

The project deploys local LLMs on a Snapdragon 8 Elite platform with an **Adreno 830 GPU**.

The inference stack is:

```text
Quantized GGUF Model
        ↓
     llama.cpp
        ↓
   Vulkan Backend
        ↓
   Mesa Turnip
        ↓
    Adreno 830
        ↓
 Snapdragon 8 Elite
```

The underlying Linux environment is documented separately in:

**[`duo-systems-on-snapdragon-8-elite-tablet`](../duo-systems-on-snapdragon-8-elite-tablet)**

## Hardware

- **SoC:** Qualcomm Snapdragon 8 Elite
- **CPU architecture:** ARM64
- **GPU:** Adreno 830
- **RAM:** 16 GB
- **Storage:** 512 GB
- **GPU memory model:** Unified system memory

## Software

- Ubuntu 24.04 ARM64
- Termux
- Termux:X11
- Mesa
- Turnip Vulkan driver
- Vulkan 1.4
- llama.cpp
- GGUF quantized models

## Why This Project?

Running an LLM locally on a mobile SoC presents several interesting engineering constraints:

- Limited memory compared with desktop workstations
- Shared CPU/GPU memory
- Mobile GPU architecture
- ARM64 instruction set
- Mobile thermal and power constraints
- Vulkan driver compatibility
- GPU kernel compatibility
- Model quantization

The project investigates how these constraints affect real-world local inference.

## GPU Setup

The Adreno 830 is exposed to Linux through KGSL and accelerated using Mesa's Turnip Vulkan driver.

The working Vulkan configuration reports:

```text
Device:
Adreno (TM) 830v1

Driver:
turnip Mesa driver

Mesa:
26.3.0-devel

Vulkan:
1.4.362
```

Verification:

```bash
vulkaninfo --summary
```

Expected:

```text
deviceName = Adreno (TM) 830v1
driverName = turnip Mesa driver
```

## llama.cpp

The project uses llama.cpp as the local inference engine.

llama.cpp supports Vulkan as a GPU backend and provides device enumeration through:

```bash
./llama-cli --list-devices
```

The Snapdragon platform appears as:

```text
Vulkan0: Adreno (TM) 830v1
```

llama.cpp's Vulkan backend can offload model computation to the GPU, while CPU/GPU hybrid execution can also be used for models that exceed available GPU memory. citeturn0search0turn0search4

## Building llama.cpp

Build llama.cpp with Vulkan support:

```bash
cmake -B build -DGGML_VULKAN=ON
cmake --build build --config Release -j$(nproc)
```

Check GPU detection:

```bash
./build/bin/llama-cli --list-devices
```

The official llama.cpp documentation also recommends verifying Vulkan with `vulkaninfo` before building the Vulkan backend. citeturn0search0

## Running a Model

Example:

```bash
./build/bin/llama-cli \
    -m /path/to/model.gguf \
    -ngl 99 \
    -c 8192 \
    -cnv
```

Where:

- `-m` specifies the GGUF model
- `-ngl 99` attempts to offload the model layers to the GPU
- `-c` specifies the context length
- `-cnv` enables conversation mode

The optimal settings depend on model architecture, quantization, context length, and available memory.

## Model Quantization

Quantization is important for mobile inference because it significantly reduces model memory requirements.

Models tested or investigated include quantized models in formats such as:

- Q4_K_S
- Q4_K_M
- Q5
- Other GGUF quantizations

A smaller quantized model allows more of the model to remain GPU-resident while leaving memory available for:

- KV cache
- runtime buffers
- operating system
- desktop environment
- other applications

## Memory

The Vulkan device exposes approximately:

```text
11.26 GiB device-local memory
```

However, this is **unified memory**, not dedicated VRAM.

The available Vulkan memory budget is dynamic.

For this reason, model compatibility should not be determined solely from the nominal 11.26 GiB heap size.

A model's:

```text
weights
+ KV cache
+ temporary buffers
+ runtime overhead
+ operating system usage
```

must all fit within the available memory budget.

## Performance

Performance is measured using llama.cpp's inference statistics.

Example output:

```text
Prompt:      XX.X t/s
Generation:  XX.X t/s
```

The project evaluates:

- Prompt processing speed
- Token generation speed
- GPU utilization
- Memory usage
- Model size
- Quantization level
- Context length

### Benchmark Table

| Model | Quantization | Context | GPU Offload | Prompt (t/s) | Generation (t/s) |
|---|---|---:|---:|---:|---:|
| Qwen3.5 9B | Q4 | TBD | Vulkan | TBD | TBD |
| Ornith 1.5 9B | Q4_K_M | TBD | Vulkan | TBD | TBD |

> Benchmark values should be recorded from repeatable runs under controlled conditions.

## MTP / Speculative Decoding

The project also investigates newer inference techniques such as **Multi-Token Prediction (MTP)** and speculative decoding.

These techniques are particularly interesting on mobile hardware because reducing the amount of sequential decoding work can potentially improve generation throughput.

Compatibility depends on the model architecture, GGUF conversion, llama.cpp version, and backend support.

## Experiments

Current experiments include:

### 1. Vulkan GPU Acceleration

Successfully brought up:

```text
Adreno 830
      ↓
Mesa Turnip
      ↓
Vulkan
      ↓
llama.cpp
```

### 2. Quantized LLM Inference

Tested quantized ~9B-class models directly on the Snapdragon platform.

### 3. ARM64 Linux

The entire inference environment runs natively on ARM64 Linux rather than relying on x86 emulation.

### 4. Mobile GPU Memory

Investigated the relationship between:

- Vulkan device-local memory
- Vulkan memory budget
- Unified system memory
- llama.cpp GPU allocation

## Results

The project demonstrates that a phone-class SoC can be used as a **standalone local LLM inference platform** when combined with:

- Model quantization
- GPU offloading
- Vulkan
- An appropriate mobile GPU driver
- ARM64-native software

The platform successfully runs llama.cpp with the Adreno 830 exposed as a Vulkan compute device.

## Limitations

The system is not equivalent to a desktop discrete GPU.

Important limitations include:

- Shared CPU/GPU memory
- Dynamic memory budget
- Mobile thermal constraints
- Vulkan driver maturity
- Backend-specific kernel limitations
- Limited memory compared with high-end desktop GPUs

Large models may require CPU/GPU hybrid inference or lower-bit quantization.

## Future Work

Potential future experiments:

- MTP benchmarking
- Speculative decoding
- Larger models
- Q4 vs Q5 vs Q6 comparisons
- Context-length scaling
- CPU vs GPU inference
- Vulkan kernel optimization
- Power-efficiency measurements
- Sustained-performance testing
- Thermal throttling analysis
- Comparison with desktop GPUs

## Related Project

The underlying Snapdragon 8 Elite Linux platform is documented separately:

**[`Duo Systems on Snapdragon 8 Elite Tablet`](../duo-systems-on-snapdragon-8-elite-tablet)**

That project covers:

- Hardware
- Android/Linux integration
- Ubuntu
- Termux
- Termux:X11
- XFCE
- Mesa
- Turnip
- Vulkan

This repository focuses specifically on **local LLM inference**.

## Technologies

- Qualcomm Snapdragon 8 Elite
- Adreno 830
- ARM64
- Ubuntu 24.04
- Termux
- Mesa
- Turnip
- Vulkan
- llama.cpp
- GGUF
- C/C++
- Linux

## License

This repository contains configuration, documentation, benchmarks, and experimental scripts.

Model files are not included. Refer to the respective model licenses before downloading or redistributing models.