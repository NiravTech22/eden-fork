<!--
# SPDX-FileCopyrightText: Copyright 2025 Eden Emulator Project
# SPDX-License-Identifier: GPL-3.0-or-later

# SPDX-FileCopyrightText: 2018 yuzu Emulator Project
# SPDX-License-Identifier: GPL-2.0-or-later
-->
<!-- lang: en-GB -->

<h1 align="center">
  <br>
  <a href="https://git.eden-emu.dev/eden-emu/eden"><img src="./dist/qt_themes/default/icons/256x256/eden.png" alt="Eden" width="200"></a>
  <br>
  <b>Eden</b>
  <br>
</h1>

<h4 align="center"><b>Eden</b> is a free and open-source (FOSS) Switch 1 emulator started by developer Camille LaVey.
<br>
Written in C++, with builds for Windows, Linux, macOS, Android, FreeBSD and more.
</h4>

<p align="center">
    </a>
    <a href="https://discord.gg/HstXbPch7X">
        <img src="https://img.shields.io/discord/1367654015269339267?color=5865F2&label=Eden&logo=discord&logoColor=white"
            alt="Discord">
    </a>
    <a href="https://stt.gg/qKgFEAbH">
        <img src="https://img.shields.io/revolt/invite/qKgFEAbH?color=d61f3a&label=Stoat"
            alt="Stoat">
    </a>
</p>

<p align="center">
  <a href="#compatibility">Compatibility</a> |
  <a href="#development">Development</a> |
  <a href="#building">Building</a> |
  <a href="#download">Download</a> |
  <a href="#support">Support</a> |
  <a href="#license">License</a>
</p>

## This fork: purpose and progress

This personal fork explores practical improvements to Eden's memory handling and Vulkan rendering, with Windows as the primary development target. The intended hardware includes an RTX 3050 laptop with 4 GB of VRAM and a desktop GPU with 8 GB of VRAM. The aim is better image quality and consistent frame times within those limits, while preserving existing filters, accuracy settings, and support for other GPUs and operating systems.

The planned graphics work is optional NVIDIA DLSS Super Resolution through NGX Vulkan: reconstruct a low-resolution 3D scene for presentation at up to 4K, eventually keeping UI outside the temporal upscaler. This is a development target, not an available feature. The audited checkout has no DLSS integration or custom CUDA rendering enhancement; its CUDA support is associated with FFmpeg hardware video decoding.

### How problems are approached

The approach is to trace a concrete failure through the relevant subsystem, fix the underlying assumption, and keep each change small enough to review and compare. For example, commit `86ba056b6c` addresses extended-memory capacity and profile changes:

- Device-memory tracking now derives its capacity from the actual backing allocation instead of assuming every extended-memory configuration fits an 8 GiB allocation.
- System reinitialization compares the complete memory-layout setting, so switching between extended-memory profiles is detected.
- The kernel memory-layout maximum increases from 8 GiB to 12 GiB to accommodate the larger configuration. This is guest-memory capacity, not GPU VRAM.

These are concrete correctness improvements visible in the code. Their runtime behavior still needs validation in this checkout, and they do not establish an FPS improvement. No reproducible before/after performance measurements were found during the Phase 0 audit; claims of significant speedups will be added only with the game, scene, hardware, settings, and measurements that support them.

For DLSS, the same process starts with an architecture audit and a working baseline, then proves SDK initialization and synthetic inputs before attempting scene detection, jitter, or motion vectors. The performance acceptance target is under approximately 3 ms for DLSS evaluation plus added passes on the 8 GB desktop at 1080p input to 4K output, with the NGX-selected quality/input combination recorded. That target has not been measured or achieved. Resource accounting and a low-VRAM path are required throughout.

See the [Phase 0 architecture and gap analysis](docs/dlss/ARCHITECTURE.md) and [repeatable validation plan](docs/dlss/TESTING.md) for evidence, blockers, and next steps. DLSS SDK binaries and `nvngx_dlss.dll` must not be committed; the future build option will default to OFF and unavailable runtime support must fall back cleanly.

## Compatibility

The emulator is capable of running most commercial games at full speed, provided you meet the necessary hardware requirements.

A list of supported games will be available in future. Please be patient.

Check out our [website](https://eden-emu.dev) for the latest news on exciting features, monthly progress reports, and more!

[![Packaging status](https://repology.org/badge/vertical-allrepos/eden-emulator.svg)](https://repology.org/project/eden-emulator/versions)

## Contribute

To contribute to Eden; be it financially, code, bug reports, or otherwise, see our [Contributing guidelines](./CONTRIBUTING.md).

## Documentation

We have a user manual! See our [User Handbook](./docs/user/README.md).

## Building

See the [General Build Guide](docs/Build.md)

For information on provided development tooling, see the [Tools directory](./tools)

## Download

You can download the latest releases from [our release page](https://git.eden-emu.dev/eden-emu/eden/releases).

Save us some bandwidth! We have [mirrors available](./docs/user/ThirdParty.md#mirrors) as well.

## License

Eden is licensed under the GPLv3 (or any later version). Refer to the [LICENSE.txt](https://git.eden-emu.dev/eden-emu/eden/src/branch/master/LICENSE.txt) file.

## Special thanks

Super special thanks to Cloudflare for preventing the git server from blowing up.

- Yuzu
- Ryujinx
- Sudachi
- Citron
- Torzu
- Suyu
- Ryubing

And everyone who continues or had contributed to the project! <3
