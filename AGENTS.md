# MNN Project Instructions

This repository is an **Android-only** fork of Alibaba's MNN, trimmed down to what's needed to build the `MnnLlmChat` app (an offline, on-device LLM/Diffusion chat client — target: an advanced mobile analog of LM Studio). MNN itself is a lightweight deep learning **inference engine** (not a training framework). Code must prioritize **performance and binary size**.

Platform-only pieces (iOS/HarmonyOS/desktop CUDA-TensorRT-Metal-CoreML backends, PyMNN, docs, cross-compile tooling, unrelated Android demo apps) were removed. See `apps/Android/MnnLlmChat/` for the app itself.

## Architecture Overview

MNN uses a **graph optimization + heterogeneous backend scheduling** architecture.

Two inference APIs are available:
- **Session API** (low-level): `Interpreter → createSession → runSession`, operates on Tensor directly
- **Module API** (high-level, recommended): `Module::load → onForward(VARP)`, Express-based dynamic graph. Used by LLM / Diffusion and most modern workloads

**Key abstractions** (see corresponding headers under `source/core/`):
- **Interpreter** / **Session**: model loading and inference session management
- **Backend** / **Execution**: hardware backend abstraction and per-op implementation (CPU/ARM82/OpenCL/Vulkan/NNAPI/QNN/Neuropilot/HiAI — the Android-relevant backends kept in this fork)
- **Tensor**: data container; internally uses NC4HW4 format (channels packed by 4 for SIMD)
- **Op / Schema**: FlatBuffers-defined operator descriptors (`schema/default/*.fbs`)

**Op registration pattern**: Schema definition → shape inference (`source/shape/`) → Geometry decomposition (optional) → Backend Execution implementation

## LLM Subsystem

MNN supports end-to-end LLM export and inference:

- **Python export** (`transformers/llm/export/`): HuggingFace model → MNN format. Core modules: `llmexport.py` entry point, `utils/model_mapper.py` (model field mapping), `utils/model.py` (unified LlmModel class), `utils/transformers.py` (Attention/Decoder/RoPE export)
- **C++ inference** (`transformers/llm/engine/`): `llm.cpp` (text inference), `omni.cpp` (multimodal: vision/audio), includes KVCache management and sampling strategies

## Repository Structure

| Directory | Description |
|-----------|-------------|
| `include/MNN/` | Public C++ headers |
| `source/core/` | Inference core (Interpreter, Session, Pipeline, Backend) |
| `source/backend/` | Android-relevant hardware backends: cpu, arm82, opencl, vulkan, nnapi, qnn, neuropilot, hiai, opengl |
| `source/shape/` | Shape inference |
| `source/geometry/` | Geometry computation (op decomposition) |
| `express/` | Express API (high-level dynamic graph, VARP) |
| `schema/default/` | FlatBuffers schema (op definitions) |
| `tools/converter/` | Model converter (ONNX/TF/Caffe → MNN), desktop-only, used to prepare models offline |
| `tools/audio/`, `tools/cv/` | Audio/image ops linked into the Android runtime |
| `transformers/llm/` | LLM export (Python) + inference engine (C++) |
| `transformers/diffusion/` | Diffusion model support |
| `project/android/` | NDK/CMake build scripts producing `libMNN.so` for the app |
| `apps/Android/MnnLlmChat/` | The Android chat app (JNI bridge in `app/src/main/cpp/`) |
| `apps/frameworks/mnn_tts/`, `apps/frameworks/model_downloader/` | Gradle modules the app depends on |
| `skills/` | AI Agent Skills |

Removed from upstream MNN (not relevant to this Android-only fork): `pymnn/`, `test/`, `docs/`, `doc/`, `demo/`, `benchmark/`, `codegen/`, `package_scripts/`, `project/ios/`, `project/harmony/`, `project/cross-compile/`, desktop-only backends (`cuda`, `tensorrt`, `coreml`, `metal`, `musa`), `apps/mnncli`, `apps/sana`, `apps/Android/MnnTaoAvatar`, `apps/Android/Mnn3dAvatar`, `apps/frameworks/sherpa-mnn` (source — the app links a prebuilt `.so` downloaded from CDN instead).

## Coding Style

- **C++**: Google Style variant, see `.clang-format`. 4-space indent, 120-char line width, attached braces. Class names `PascalCase`, functions `camelCase`, member variables `mCamelCase`. RTTI and exceptions disabled (`-fno-rtti -fno-exceptions`). Default standard: C++11.
- **Python**: Standard Python conventions
- **Formatting**: `clang-format -i -style=file <file>`

## Build & Test

```bash
# Build the native libMNN.so for the Android app (arm64-v8a)
cd apps/Android/MnnLlmChat
./build.sh          # builds project/android/build_64 then ./gradlew assembleStandardDebug

# Common CMake options set by build.sh: MNN_BUILD_LLM, MNN_ARM82, MNN_OPENCL,
# MNN_BUILD_DIFFUSION, MNN_BUILD_AUDIO, MNN_QNN. Full list: option() declarations
# at the top of CMakeLists.txt (MNN_BUILD_TEST/MNN_BUILD_BENCHMARK are forced OFF
# in project/android/build_64.sh since test/ and benchmark/ were removed from this fork)

# LLM export (desktop, prepares a model to load into the app)
cd transformers/llm/export
python llmexport.py --path /path/to/model --export mnn --hqq --dst_path ./MODEL
```

There is no C++ unit-test suite in this fork (`test/`, `benchmark/` were removed as out of scope for the Android-only app). Validate changes by building and running `MnnLlmChat` on-device/emulator.

## Commit Message

One-line English summary with a `[Module:Type]` prefix. Module: `LLM`, `CPU`, `Metal`, `CUDA`, `OpenCL`, `Vulkan`, `Core`, `Infra`, `Doc`, etc. Type: `Feature`, `Bugfix`, `Perf`, `Refact`, `Style`, `Doc`, `Test`, `Chore`, `Release`.

Example: `[LLM:Feature] Add streaming support`

## Skills

For the following tasks, **read the Skill entry file first** and execute step by step. Each step must pass its tests before proceeding.

**After non-trivial skill-driven tasks, run Retrospective only when there are reusable lessons.**

Public skills are listed below. Environment-dependent skills may exist under `skills/*/SKILL.md`.

| Skill | Entry File | Trigger |
|-------|-----------|---------|
| Support new LLM | `skills/support-new-llm/SKILL.md` | Add / adapt a new LLM model |
| Add new op | `skills/add-new-op/SKILL.md` | Add a new operator |
| ARM CPU optimization | `skills/arm-cpu-optimize/SKILL.md` | Optimize op performance on ARM CPU |
| OpenCL optimization | `skills/opencl-optimize/SKILL.md` | Optimize op performance on OpenCL |
| Vulkan optimization | `skills/vulkan-optimize/SKILL.md` | Optimize op performance on Vulkan |
| Retrospective | `skills/retrospective/SKILL.md` | After non-trivial tasks with reusable lessons |
