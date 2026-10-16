# DLSS validation plan

Status on 2026-10-16, Phase 0R: source audit re-verified; baseline configuration still blocked because CMake is not visible. DLSS/CUDA enhancements are absent. Hardware enumeration found an RTX 3050 8192 MiB, driver 617.14, compute capability 8.6; Vulkan loader instance version is 1.4.341. Khronos validation is not listed. All emulator runtime, validation-layer, timing, image-quality, and VRAM-delta tests below are **NOT RUN**. See [PHASE_0R.md](PHASE_0R.md) for actual commands/results.

## Establish the build baseline

1. Confirm the intended checkout/branch and save `git rev-parse HEAD` and `git status --short`. Phase 0R started at documentation commit `d6a61eb2ee` on `backup/extended-memory-2026-10-13`; implementation remains at `86ba056b6c`. Run `git submodule update --init --recursive`; in this checkout it succeeds with no submodules to initialize.
2. Follow `docs/Deps.md` and `docs/Build.md`. Make CMake 3.31+, Ninja, the MSVC x64 toolchain or clang-cl environment, Python, and required dependencies available. Use `tools/windows/load-msvc-env.ps1` in the build shell after Visual Studio is installed. Record compiler, CMake, dependency versions, GPU, driver, Windows version, power mode, and display refresh rate.
3. Run from the repository root and retain full configure/build logs:

```powershell
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DYUZU_TESTS=OFF -DYUZU_ENABLE_DLSS=OFF -DYUZU_ENABLE_CUDA_ENHANCEMENTS=OFF
cmake --build build
```

Only run the build after configuration succeeds. These use the Windows command-line guide's configure flags plus the two explicitly requested OFF variables. The Phase 0R attempt failed at command lookup in 0.266 seconds; compilation never ran, so build duration is unavailable. Both new variables are currently undeclared and would not test real feature gating even if configuration succeeded. Use a suitable installed Qt/dependency configuration if defaults cannot resolve it, and record every extra option. Report the first actual build error before changing source to bypass it.

## ON/OFF matrix once the option exists

Use separate clean build trees with the same compiler/dependency configuration for (DLSS, CUDA) = (OFF, OFF), (ON, OFF), (ON, ON), and (OFF, ON) for the standalone interop self-test. Both options must default OFF. These are future options, not implemented gates in the audited baseline. Determine SDK discovery parameters from the actual implementation before configuring ON. Do not commit SDK libraries, DLLs, or CUDA toolkit files. Query `nvidia-smi --query-gpu=compute_cap --format=csv,noheader` on each target and verify supported compiler targets instead of hardcoding an architecture.

Cover Windows MSVC and clang-cl; compile Linux with OFF. Confirm OFF has no SDK discovery requirement, link dependency, runtime load, extension additions, or new frame resources. On an ON build, test runtime disabled, supported RTX with runtime available, missing runtime, unsupported GPU, insufficient driver, invalid dimensions, and initialization/evaluation failure. Every failure must log a reason and preserve a working fallback.

## Runtime validation and captures

Use a Vulkan SDK installation with Khronos validation and synchronization validation enabled through its layer configuration. Enable Eden renderer debug logging as well; that setting alone is not proof that synchronization validation is active. Save layer configuration and startup logs showing the active layers/features. Exact setup must be verified against the installed SDK version before execution.

For each case, retain complete logs from startup through shutdown. Require zero validation/synchronization errors; track warnings and attribution separately. Exercise resizing, fullscreen changes, quality changes, pause/resume, loading screens, camera cuts, guest resolution changes, multiple layers, screenshots, and applet capture. Verify resource release and feature recreation without use-after-free or new hot-path idle waits.

Use repeatable saves and camera paths, warm caches first, then record at least three 60-second runs per configuration. Keep CPU/GPU accuracy, speed limit, mods, guest resolution, refresh rate, power mode, and other filters fixed. Capture these configurations separately:

- Native at output resolution, where supported, as a reference.
- Existing FSR with the same low-resolution input used for DLSS.
- DLSS synthetic inputs (Phase 1), explicitly labeled as plumbing validation.
- Later DLSS with each motion-vector mode and quality preset, logging actual SDK-selected input dimensions.

Record shimmer on thin geometry, disocclusion ghosting, transparent effects, particles, UI/text blur, color/exposure changes, and camera-cut recovery. Use matching screenshots plus motion video; a still image cannot establish temporal stability. RenderDoc captures should show attachment selection, jitter isolation, layouts, and pass order; capture tooling compatibility with the chosen NGX runtime remains to be checked. Do not use capture-instrumented runs as performance results.

## Proposed game/scene matrix

These are suggested locally owned test workloads, not games verified in this audit. Record game version, title ID from the running emulator, save/route, mods, and exact scene; do not invent title IDs.

| Game / scene | Coverage |
| --- | --- |
| The Legend of Zelda: Breath of the Wild, outdoor foliage with slow pan and HUD | Thin geometry, foliage, camera motion, UI contamination |
| Super Mario Odyssey, traversal in a populated outdoor kingdom and menu transition | Animated characters, disocclusion, particles, scene transitions |
| Mario Kart 8 Deluxe, fixed time-trial route with HUD and effects | Fast motion, transparency, stable repeatable route |
| A locally available 2D/menu-heavy title | Scene-classifier rejection/fallback and crisp text |

Start with one reproducible 3D-heavy scene on each target GPU; broaden only after stable plumbing. Record classifier decisions and failure captures before adding title-specific overrides. Reprojection relying on known constant-buffer layouts or draw matching must be labeled game-specific.

## Timing, latency, and VRAM

Once implemented, bracket every added conversion/copy, depth pass, optical-flow pass, MV conversion, and DLSS evaluation with GPU timestamps in the existing command stream. Verify the timestamp API and queue support against actual Vulkan headers/spec before coding. Read completed results asynchronously; do not block the current frame for profiling. Record timestamp period/valid-bit handling and unsupported timing explicitly.

Report per-pass and total added GPU time (median, p95, p99), frame-time variance, dropped/presented frames, CPU submission timing, and queue occupancy. Test FIFO, mailbox, immediate, and relaxed FIFO where available, with asynchronous presentation on/off and speed-limit/turbo combinations. Super Resolution must not introduce generated frames or increase the existing frame pool. Timestamps measure GPU duration, not end-to-end input latency; use input-to-display measurement for that claim.

Acceptance target: approximately 3 ms maximum target budget for evaluation plus added passes on the 8 GB desktop in the requested 1080p-to-4K workload. Also test the SDK-recommended Quality dimensions and identify the two workloads separately if they differ. The detected local 8 GB GPU is an NVIDIA GeForce RTX 3050; the separate 4 GB/30 W laptop was not queried. VRAM size alone does not establish compute performance.

Log allocation bytes/count for every new history, color/depth/MV image and scratch allocation, plus SDK-reported internal usage if available. Record unknown internal costs separately, baseline/peak dedicated-memory use, and sustained paging/stutter on the 4 GB laptop. Use the arithmetic estimates in ARCHITECTURE.md only for planning. Test Balanced/Performance and output-resource reuse as low-VRAM candidates after SDK verification.

## CUDA interop and compute comparison gates

Before Phase 2, locate real CUDA, Vulkan, and Optical Flow headers/docs. Establish adapter identity using verified LUID/UUID APIs; do not infer a match from GPU ordinal or name. Verify external image/buffer handle support and ownership plus binary/timeline semaphore capabilities for the actual driver/API combination. Document the implemented sync model in INTEROP.md when it exists; no interop is implemented now.

Self-test both directions: CUDA writes a shared gradient sampled by Vulkan, and Vulkan writes a pattern read by CUDA. Compare readback against expected values and stress resource reuse/recreation. Readback is for the isolated correctness test, not the steady-state frame loop. Validate GPU-side semaphore ordering without device/queue idle calls or CUDA device synchronization. Stop for review if measured handoff overhead exceeds approximately 0.5 ms or introduces CPU stalls.

For MV upsampling/conversion, luminance/MV reduction, and any necessary depth transform, benchmark a matching Vulkan compute shader against each CUDA kernel. Use identical data, format/precision, warmup, dimensions, and repeated runs. Measure CUDA execution with verified event APIs, Vulkan execution with timestamps, and whole-frame impact separately; do not subtract unrelated CUDA/Vulkan clock domains. Include handoff costs and SDK work rather than comparing kernel time alone. Read timing results after completion without blocking the current frame.

Report median/p95/p99, correctness tolerance, bytes allocated/shared, and end-to-end delta. Retain CUDA only if it wins or enables a verified capability unavailable through the Vulkan route. If it loses, recommend its removal. Current results for every kernel and interop case: NOT RUN; no kernels exist. Extend the VRAM ledger to CUDA staging and SDK allocations, avoiding double-counting imported shared payloads.

Restart phase numbering: 1 NGX synthetic inputs; 2 CUDA interop; 3 scene classification; 4 jitter; 5 motion/depth/reset and compute comparisons; 6 pacing; 7 UX. Each phase stops with a report before advancing.

## Regression checklist and phase report

- OFF build and runtime-disabled ON build preserve FSR, SGSR, SMAA, FXAA, ReShade, screenshots, and applet capture.
- AMD/Intel exercise startup, existing filters, swapchain recreation, and clean DLSS unavailability. Until hardware/CI is available, mark these as source reasoning only, never PASS.
- CPU/GPU accuracy, internal resolution scaling, guest video decoding, and shader caches retain their prior behavior.
- Temporal resources reset on lifecycle/discontinuity changes; shader variants cannot contaminate existing caches.
- No SDK binary appears in the Git diff; Linux OFF still configures/builds.

Every phase report must include files/commit, exact commands and results, validation logs, capture comparisons, per-pass timing, VRAM delta, failures/unknowns, and the next gate. Phase 0 adds no runtime resources or passes; its runtime delta is structurally zero, not a measured benchmark. No performance comparison is available yet.
