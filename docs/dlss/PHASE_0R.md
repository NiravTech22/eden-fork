# Phase 0R: environment, repository truth, and baseline attempt

Date: 2026-10-16. Checkout: `C:\Users\Nirav\Documents\GitHub\eden-fork`. Starting HEAD: `d6a61eb2ee`. No feature code, CMake options, toolkit files, or SDK binaries were added. The existing documentation commit is retained. The owner explicitly confirmed restarting here from scratch.

## Repository truth

Commands run from the checkout:

```powershell
git status --short
git branch -a
git stash list
git reflog -30
git log --all --oneline -i --grep=cuda
git grep -il cuda -- ':!*.md'
git grep -n -i -E 'ngx|dlss|streamline' -- ':!*.md' ':!*.svg' ':!*.xml'
git grep -n -E 'YUZU_ENABLE_DLSS|YUZU_ENABLE_CUDA_ENHANCEMENTS|NVSDK_NGX|cudaImportExternal|NvOF|Streamline' -- CMakeLists.txt CMakeModules src externals
git log --all --oneline -i --grep=dlss --grep=ngx --grep=streamline
git submodule update --init --recursive
git submodule status
git ls-files .gitmodules
```

- Starting status: clean. Local branch: `backup/extended-memory-2026-10-13`. Only remote branch shown: `origin/backup/extended-memory-2026-10-13`, also the remote HEAD. These are locally known refs; no remote fetch was performed.
- Stash list: empty. Reflog has two entries: `d6a61eb2ee` documentation commit, then `86ba056b6c` clone. No recoverable feature work appeared in these checks. Unreferenced objects, other clones, and unsaved editor buffers were not searched; the restart does not depend on recovering them.
- CUDA commit-message matches: `0102f548d0`, `a8564a09b7`, `c4e1a9a7a6`, `9d3cce87a4`, `42efbdc492`, `168882cb7a`, `f91cc356fb`, `5db0df833a`, `0c3acedaf9`. Subjects concern FFmpeg/video decoding or guest shader translation, not a recovered CUDA enhancement pipeline.
- CUDA source matches: `externals/ffmpeg/CMakeLists.txt` (optional CUDA discovery/link support); `src/video_core/host1x/ffmpeg.cpp` (CUDA hardware-decoder enum entries); three shader translator files (`barrier_operations.cpp`, `load_store_memory.cpp`, `surface_load_store.cpp`) containing CUDA/PTX comments. The receive-frame path can transfer a hardware-decoded frame through FFmpeg. It is not a Vulkan-CUDA shared-scene pipeline.
- Other substring matches are false positives: Turkish `sunucudan` in `dist/languages/tr_TR.ts`, and Joy-Con `MCUData` names in `common_protocol.cpp`, `common_protocol.h`, and `nfc.cpp`.
- NGX/DLSS/Streamline source substring matches were certificate text in `.patch/openssl/0001-add-bundled-cert.patch` and HID `Hdls` names. The focused feature/option grep produced no matches. History search found the previous audit plus ordinary English uses of "streamline" (`728de0406e`, `e2647971f8`, `427eca063d`, `e97cd00753`), not NVIDIA Streamline integration.
- Submodule update returned exit 0. Submodule status, `.gitmodules` inventory, and index mode-160000 search were empty: there are no tracked submodules to populate. Externals use package declarations; this command does not download them.

Conclusion: no NGX/DLSS or CUDA support-kernel/interop implementation exists in the inspected tree. Existing FFmpeg support is not being adopted as the new enhancement implementation; no feature work was built upon it.

## Tools and hardware

`Get-Command cmake,ninja,cl,nvcc,vulkaninfo,nvidia-smi -ErrorAction SilentlyContinue` resolved only the two tools in `C:\Windows\System32`. No CUDA/Vulkan/VS/CMake environment variables matched the inspected environment. This process is PowerShell and does not inherit the user's described Native Tools environment.

| Tool | Observed result |
| --- | --- |
| CMake | Not on PATH; version unavailable |
| Ninja | Not on PATH; version unavailable |
| MSVC cl | Not on PATH; version unavailable |
| nvcc | Not on PATH; toolkit version unavailable |
| vulkaninfo | Runs; reports Vulkan instance version 1.4.341 and GPU API 1.4.351; executable file/product version metadata was empty |
| nvidia-smi | `--version` reports NVIDIA-SMI/NVML/KMD 617.14 and CUDA UMD 13.4; UMD support is not proof of an installed CUDA compiler/toolkit |

The standard CMake, Visual Studio/vswhere, CUDA Toolkit, Qt, and Vulkan SDK locations checked did not exist. Targeted executable/header inventories under Program Files, Program Files (x86), user-local Programs, the user tree, and D: returned no matching build tools or SDK headers. Some inaccessible paths were suppressed in those inventories; this is not proof that no nonstandard installation exists anywhere. No installation/removal was attempted.

Hardware command:

```powershell
nvidia-smi --query-gpu=name,memory.total,driver_version,compute_cap --format=csv
nvidia-smi --version
vulkaninfo --summary
```

Detected GPU: **NVIDIA GeForce RTX 3050**, **8192 MiB**, driver **617.14**, compute capability **8.6**. Candidate architecture for this local device is 8.6; no architecture setting or build was introduced. Toolkit compatibility must be verified before selecting compiler flags. Vulkan reports the same model, vendor `0x10de`, device `0x2507`, UUID `68110f7b-1232-0812-bb23-40f678cb3126`. A CUDA identity query has not run, so this is not an interop adapter-match test. The separate 4 GB/30 W laptop remains untested.

Vulkan reports four layers: NVIDIA Optimus, NVIDIA Presentation, Steam Fossilize, and Steam Overlay. **Khronos validation is absent from this enumeration.** Loader diagnostics report a missing Epic Online Services `EOSOverlayVkLayer-Win64.json` and a registry layer-manifest lookup warning. Enumeration still prints device information. This is not clean emulator validation and no registry/overlay changes were made.

## Baseline attempt

Read `docs/Build.md`, `docs/Deps.md`, root CMake declarations, and `tools/windows/load-msvc-env.ps1`. The guide's Windows configure command uses Ninja, Release, and `YUZU_TESTS=OFF`; the root requires CMake 3.31+. The environment helper requires vswhere, which was not found at its expected path.

Exact attempted configure command, using the guide flags plus the requested OFF values:

```powershell
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DYUZU_TESTS=OFF -DYUZU_ENABLE_DLSS=OFF -DYUZU_ENABLE_CUDA_ENHANCEMENTS=OFF
```

Elapsed command attempt: **0.2659974 seconds** via `System.Diagnostics.Stopwatch`. First error: **The term 'cmake' is not recognized as the name of a cmdlet, function, script file, or operable program.** No CMake process started; there is no CMake exit code. Compilation did not run; build time is **N/A**, not zero or 0.266 seconds. The documented subsequent default-target command is `cmake --build build` and was not run because configuration never began.

Both new CMake options are absent, so supplying OFF cannot yet prove compile-time gating. No source error was reached or fixed. No ON/OFF build, Linux configure, shader compilation, or SDK API check has passed. The missing-tool result conflicts with the user's reported installation; obtain the installation paths or a session inheriting the configured environment before proceeding.

## Source re-verification and corrections

Re-read/grepped Maxwell draw dispatch, GPU/display composition, Vulkan rasterizer, graphics pipeline configuration, shader translation/SPIR-V emission, texture-cache target/scaling selection, Vulkan image/framebuffer creation, layer/window-adapt passes, scheduler, present manager, swapchain, instance/device setup, allocator, CMake dependencies, and FFmpeg receive handling.

The main architecture claims from `d6a61eb2ee` remain supported. Corrections/additions:

- Expanded presentation filter coverage beyond FSR/SGSR and AA.
- Explicitly documented existing device-idle waits on presentation resource recreation; the renderer is not globally wait-free.
- Replaced the obsolete missing-checkout prerequisite with the confirmed restart decision.
- Added actual GPU/tool observations, including the distinction between Vulkan runtime availability and installed SDK/validation layers.
- Added CUDA-only self-test configuration, bounded CUDA responsibilities, mandatory equivalent Vulkan benchmarks, and the interop overhead gate.
- Clarified separate CUDA-stream execution joined by GPU semaphore dependencies. A single Vulkan command buffer cannot literally contain the proposed CUDA kernels.

Resource payload estimates remain arithmetic planning values, not verified SDK formats or allocations. No new runtime resources or passes exist, so there is no measured VRAM delta or timing result. Native/FSR/DLSS screenshots and non-NVIDIA regression execution remain NOT RUN.

## Ordered blockers and next gate

1. Locate/expose CMake, Ninja, and the MSVC environment; complete the unchanged baseline build and report its first real error before source changes.
2. Locate the NGX, CUDA, Optical Flow, and Vulkan SDK headers/docs; establish validation-layer availability. No API signatures or OF backend support are assumed.
3. Verify NGX capability/loading/extension ordering and resource state/lifetimes before synthetic-input integration.
4. Verify adapter identity/external-resource interoperability and compare CUDA overhead to the 0.5 ms gate before support kernels.
5. Scene/UI separation, depth conventions, jitter isolation, temporal reset, and motion vectors remain unsolved. Per-title target overrides or constant-buffer interpretations are game-specific, not a general solution.
6. Measure the 3 ms frame budget, low-VRAM behavior, CUDA-versus-Vulkan results, and frame pacing on both machines before any performance claim.

Stop at Phase 0R. Baseline compilation is blocked, so Phase 1 is not authorized to start under the requested gate.
