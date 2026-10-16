# DLSS integration: Phase 0 architecture audit

Audit date: 2026-10-16. Source baseline: `86ba056b6c969307b0a8ab0ef04144ad2c574cee`, branch `backup/extended-memory-2026-10-13`. Local checkout: `C:\Users\Nirav\Documents\GitHub\eden-fork`. Git status was clean before this documentation change. Findings apply to this checkout, not an uninspected CUDA branch or another machine.

**Status: source audit documented; baseline build blocked by the local toolchain. No DLSS feature code added.** The repository AGENTS.md prohibits AI work; the owner's explicit request authorizes this fork audit and documentation work. No upstream submission is part of this task.

## Verified source map

Paths below are relative to the repository root. Function names are searchable anchors.

| Area | Files / anchors |
| --- | --- |
| Maxwell draw dispatch | `src/video_core/engines/maxwell_3d.cpp`; `engines/draw_manager.cpp`: `DrawEnd`, `ProcessDraw` |
| Vulkan draws | `src/video_core/renderer_vulkan/vk_rasterizer.cpp`: `PrepareDraw`, `Draw`; `vk_graphics_pipeline.cpp`: `ConfigureDraw` |
| Guest target cache | `src/video_core/texture_cache/texture_cache.h`: `RescaleRenderTargets`, `UpdateRenderTargets`, `GetFramebufferId` |
| Host images / attachments | `src/video_core/renderer_vulkan/vk_texture_cache.cpp`: `Image::Image`, `ScaleUp`, `ScaleDown`, `Framebuffer::Framebuffer` |
| Shader translation | `src/shader_recompiler/frontend/maxwell/`, `frontend/ir/`, `ir_opt/`, `backend/spirv/`; `src/video_core/renderer_vulkan/vk_pipeline_cache.cpp`: `TranslateProgram`, `EmitSPIRV`, `CreateGraphicsPipeline` |
| Guest display handoff | `src/core/hle/service/nvnflinger/hardware_composer.cpp`; `src/core/hle/service/nvdrv/devices/nvdisp_disp0.cpp`: `Composite`; `src/video_core/gpu.cpp`: `RequestComposite` |
| Final composition | `src/video_core/renderer_vulkan/renderer_vulkan.cpp`: `Composite`; `vk_blit_screen.cpp`: `DrawToFrame`; `present/window_adapt_pass.cpp`; `present/layer.cpp`: `ConfigureDraw` |
| Filters | `src/video_core/renderer_vulkan/present/{fsr,sgsr,fxaa,smaa,post_process}.cpp`; shader sources in `src/video_core/host_shaders/` |
| Submission / presentation | `src/video_core/renderer_vulkan/vk_scheduler.cpp`, `vk_present_manager.cpp`, `vk_swapchain.cpp` |
| Instance / device / allocation | `src/video_core/vulkan_common/vulkan_instance.cpp`, `vulkan_device.cpp`, `vulkan_memory_allocator.cpp` |
| CUDA references | `src/video_core/host1x/ffmpeg.cpp`; `externals/ffmpeg/CMakeLists.txt`; shader translation comments referencing CUDA terminology/PTX |

## Frame flow

1. Maxwell engine methods feed the draw manager. `DrawManager::ProcessDraw` calls the rasterizer's `Draw` for ordinary draws; indirect and other paths also exist and must be covered by future instrumentation.
2. `RasterizerVulkan::PrepareDraw` flushes pending work/caching, gets a graphics pipeline, locks buffer and texture caches, configures resources, and updates dynamic state. `Draw` records indexed/non-indexed commands into the scheduler.
3. `GraphicsPipeline` updates render targets through the shared texture cache and requests a render pass using `texture_cache.GetFramebuffer()`. Descriptor state, pipeline state, and rescaling constants are bound alongside the draw. Pipeline creation translates guest Maxwell shaders into IR and emits SPIR-V; there is no verified DLSS shader variant.
4. Color/depth views come from guest target state. `UpdateRenderTargets` prepares views, tracks active attachments, maps draw buffers, applies scaled surface dimensions, and increments a target serial when the set changes. `GetFramebufferId` caches the attachment combination; the Vulkan runtime creates backing images/views and framebuffers. The shared cache also handles aliases, feedback, copies, resolves, and scaling, so a detached scene copy must respect guest image lifetimes and later reads.
5. Guest display composition is a separate path: the hardware composer passes layers to `nvdisp_disp0::Composite`, which constructs framebuffer configurations/fences and calls `GPU::RequestComposite`. GPU coordination eventually calls `RendererVulkan::Composite`.
6. The Vulkan renderer handles applet capture/screenshots and obtains a reusable present frame. `BlitScreen::DrawToFrame` uses the window-adapt/layer path to compose guest layers into that frame.
7. `Layer::ConfigureDraw` asks `RasterizerVulkan::AccelerateDisplay` to find the guest framebuffer in the texture cache. On success it receives the image/view, original size, and scaled size. Otherwise it uploads a raw image. This interface provides color, not scene depth, motion vectors, draw identities, or camera transforms.
8. Layer processing applies FXAA/SMAA, optional ReShade processing, then FSR/SGSR where selected, and sets up the final cropped/transformed layer draw. This is already composed guest content: a game's HUD can be baked into its color target even if host display layers are separate.
9. `RendererVulkan::Composite` flushes with the frame's render-ready semaphore and calls `PresentManager::Present`. The manager acquires the swapchain image, submits a separate copy/blit from the intermediate frame, and calls `Swapchain::Present`.

The project also contains Lossless Scaling frame-generation code behind `HAS_LSFG`; its CMake option is Android-dependent. It is not NVIDIA DLSS or NVIDIA Optical Flow SDK integration.

## Resolution scaling

`Settings::UpdateRescalingInfo` derives `resolution_info` from `resolution_setup`. Texture-cache target selection uses rescalability, image history/ratings, and related attachments to decide whether to use scaled images. Surface dimensions use `(dimension * up_scale) >> down_shift`; viewport/scissor state is dirtied when scaling changes. Vulkan `Image::ScaleUp`/`ScaleDown` manages scaled backing resources, and `AccelerateDisplay` exposes the resulting dimensions.

Internal resolution scaling and the presentation filter are distinct. FSR/SGSR process the display image after guest rendering; they do not identify the game's main scene. Lowering guest rendering cost requires an appropriate internal render size, not simply adding a presentation upscaler after rendering at full output resolution. A DLSS quality label must not silently promise a particular render ratio: obtain and log the SDK-selected dimensions once real headers are available.

## Synchronization and pacing

`Scheduler::Record` queues recording work. `Flush` submits asynchronously; `Finish` submits and waits. Render-pass boundaries are mediated by `RequestRenderpass` and `RequestOutsideRenderPassOperationContext`. Layer resources already use scheduler ticks and waits for safe reuse.

`PresentManager` owns a bounded frame pool, render-ready semaphores, and present-done fences. `GetRenderFrame` waits for a free frame and its fence. Asynchronous presentation uses a worker queue/thread; synchronous presentation waits for scheduler recording and copies directly. Without LSFG, the pool is bounded by the swapchain image count and the manager's maximum. The existing separate copy submission waits on acquisition and render readiness before signaling the semaphore used for present.

`ChooseSwapPresentMode` in `vk_swapchain.cpp` supports FIFO, relaxed FIFO, mailbox, and immediate, with availability fallback. Speed-limit/turbo state affects selection; the frame-generation setting also affects it. Preserve this logic and the frame pool for DLSS Super Resolution. Do not claim zero added latency from source inspection: queue occupancy, GPU time, and frame-time variance need measurement.

The present copy currently describes the intermediate frame's producer as a color-attachment write. Writing DLSS output directly into that frame would require reviewing that dependency for the actual producer, as well as format, storage usage, and layout. Avoid an extra full-resolution staging image if supported, but reuse is not yet proven valid. SDK evaluation should be scheduled before the existing render-ready flush, outside a render pass, with explicit lifetime and state handling. No Vulkan API or NGX signature is introduced by this audit.

## CUDA, existing DLSS, and build state

The FFmpeg decoder lists CUDA among hardware decode backends on Windows/Unix. Its receive path can transfer decoded hardware frames using FFmpeg before returning them. FFmpeg's CMake contains optional CUDA discovery/link support. This is guest video decoding, not a CUDA scene-rendering, motion-vector, or DLSS pipeline. Shader comments mentioning CUDA/PTX are not host CUDA kernels.

Searches of source, CMake modules, externals, and documentation found no NGX context, DLSS option, SDK headers, feature creation/evaluation, DLSS stubs, or custom CUDA renderer implementation. No `nvsdk_ngx_vk.h` or `vulkan_core.h` was found in the inspected checkout inventory. Vulkan headers/utility libraries are declared as external dependencies in `externals/CMakeLists.txt`; `video_core` links the corresponding header targets and Vulkan Memory Allocator. They are not evidence that the actual headers have been materialized locally.

Consequently, all NGX signatures, extension helpers, initialization/loading strategy, resource conventions, and SDK-internal memory queries remain unverified. Phase 1 must inspect the actual chosen SDK headers/docs and library/export requirements before choosing runtime loading code. Do not equate the feature DLL with an assumed export surface for all NGX APIs. `VK_NVX_binary_import` is not the integration route.

### Baseline attempt

From the repository root, the exact attempted command was:

```powershell
cmake -S . -B build/dlss-audit-baseline -G Ninja -DCMAKE_BUILD_TYPE=Release
```

Result: PowerShell `CommandNotFoundException`: `cmake` is not recognized. Configuration never began; compilation was not reached. The enclosing audit command continued to read files, so its eventual process exit code does not represent successful CMake configuration.

`Get-Command cmake,ninja,cl,clang-cl,git -ErrorAction SilentlyContinue` found only Git. Standard CMake, Visual Studio installer/vswhere, Visual Studio, Qt, and Vulkan SDK locations checked were absent. A nonstandard installation may exist. The repository requires CMake 3.31+ and supplies `tools/windows/load-msvc-env.ps1`, which itself depends on vswhere. No existing build directory was present in the root inventory.

No source build defect has been demonstrated and no baseline source fix is justified yet. A usable CMake/compiler/dependency environment is required to configure, compile, and diagnose real failures. `YUZU_ENABLE_DLSS` does not exist here; setting an unused CMake variable ON/OFF would not verify either integration configuration. Linux compilation is also unverified. See [TESTING.md](TESTING.md) for continuation steps.

## Gap analysis and ordered risks

| Priority | Gap / risk | Existing foundation | Required next evidence or work |
| --- | --- | --- | --- |
| P0 | Checkout differs from described CUDA/DLSS fork | Audited branch and commit are identified | Locate intended changes or explicitly continue from this baseline |
| P0 | No working build toolchain in this session | Build/dependency guides and MSVC environment script | Successful baseline configuration/build before feature changes |
| P0 | NGX SDK/API contract absent | Vulkan extension/device setup and dynamic-library infrastructure | Inspect actual SDK headers/docs; verify loading, extension query ordering, capability checks, licensing exclusions, graceful fallback |
| P1 | No temporal inputs or scene identity | Color/depth attachment cache and scheduler; GPU draw/pass logging hooks | Synthetic-input validation first; then per-frame attachment/draw-order instrumentation |
| P1 | Scene/UI split is not generally reconstructible from final color | Guest render targets and final layer composition | Track target dependencies; assess post-FX, multiple scene passes, resolves, and reused depth; retain final-frame fallback |
| P1 | Unsafe temporal resource/state reuse could regress rendering | Tick/fence lifetime management and existing barriers | Validate formats/usages/layouts and producer-consumer dependencies, including present-copy producer changes |
| P1 | Motion vectors and jitter may be incorrect | Shader IR, position pass, pipeline keys | Verified SDK units/signs, stage coverage, variant keys, scene-only application; capability-check Optical Flow separately |
| P1 | Draw identity/reprojection is game-dependent | Guest shaders and constant buffers | Demonstrate stable identities and previous transforms per game; do not assume a universal matrix layout |
| P2 | VRAM and time budget unmeasured | Existing allocator and bounded present frames | Allocation ledger, SDK memory accounting where supported, timestamp measurements, low-VRAM modes |
| P2 | Lifecycle/settings/regression coverage missing | Existing settings and renderer lifecycle | Reset/recreate policy, title-ID overrides, missing-runtime tests, non-NVIDIA/Linux coverage |

Scene classification is a general heuristic, not a guarantee. A title-ID override that names a target/pass or interprets particular constant buffers is a game-specific workaround. Running later guest UI draws at output resolution may conflict with guest framebuffer dimensions, coordinates, shader assumptions, and later sampling; do not promise it without captures. Existing `ir_opt/position_pass.cpp` handles a particular vertex-position conversion, not a general DLSS jitter solution; geometry/tessellation and existing guest jitter need investigation too.

## Resource budget planning (not allocations or measurements)

This documentation change adds zero runtime resources. Future resource estimates below are arithmetic payload sizes for one image, excluding alignment, metadata, array layers, MSAA, duplication across in-flight frames, and SDK allocations. Formats are candidates, not verified NGX requirements.

| Candidate resource | Dimensions / bytes per pixel | Payload MiB |
| --- | --- | ---: |
| Previous scene color, RGBA8 | 1920 x 1080 x 4 | 7.91 |
| Previous scene color, RGBA16F alternative | 1920 x 1080 x 8 | 15.82 |
| Depth copy, R32F candidate | 1920 x 1080 x 4 | 7.91 |
| Motion target, RG16F candidate | 1920 x 1080 x 4 | 7.91 |
| Output, RGBA8 candidate | 3840 x 2160 x 4 | 31.64 |
| Output, RGBA16F alternative | 3840 x 2160 x 8 | 63.28 |
| OF grid/intermediates, NGX history/scratch | SDK-dependent | Unknown |

Log actual committed bytes, resource count/lifetime, render/output dimensions, peak delta, and any unobservable SDK cost explicitly as unknown. Do not present the table as total DLSS overhead. Prefer reusing compatible present output and lower render dimensions, avoiding additional full-resolution history/copies; the SDK and optical-flow resource requirements must decide the actual design. Guest 12 GiB memory support is unrelated to dedicated GPU VRAM capacity.

## Phase boundary

Phase 0 documentation is available for review, but the baseline-build gate is not satisfied. No runtime validation, screenshots, GPU timing, VRAM measurement, or significant performance gain is claimed. Resolve checkout/toolchain/SDK inputs, produce the baseline build report, and only then begin Phase 1.
