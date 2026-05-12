<div align="center">

# 🚀  FFMPEG Proxy VMix (Enable GPU Acceleration for AMD GPU's)

Transparent libx264 → h264_amf converter for AMD GPU acceleration

[![Version](https://img.shields.io/badge/version-0.1b-blue.svg)](#-changelog) [![Status](https://img.shields.io/badge/status-stable-success.svg)](#-changelog)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE) [![FFmpeg](https://img.shields.io/badge/FFmpeg-GPL%20v3-red.svg)](https://ffmpeg.org)  
[![Build](https://img.shields.io/badge/build-Linux%20%2F%20WSL2-orange.svg)](https://ubuntu.com) [![Target](https://img.shields.io/badge/target-Windows%2010%2F11%20x64-blue.svg)](https://www.microsoft.com)  
[![Donate](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-donate-FFDD00?logo=buy-me-a-coffee&logoColor=black&labelColor=white)](https://buymeacoffee.com/amazoniaaudiovisual)

[Features](#-features) • [Quick Start](#-quick-start) • [Installation](#-installation) • [Usage](#-usage) • [How it Works](#-how-it-works) • [Troubleshooting](#-troubleshooting) • [Support](#-support)

</div>

---

## 📖 What is this?

FFmpeg AMD AMF Proxy is an intelligent proxy that automatically converts CPU-based H.264 encoding (libx264) to AMD GPU-based H.264 encoding (h264_amf). It is designed for tools like vMix that don’t natively expose AMD encoders in their external-FFmpeg workflows, allowing you to keep your existing settings while offloading encoding to the GPU.

### Architecture

```
┌──────────┐         ┌─────────────┐         ┌──────────────┐
│  vMix    │ ─────→  │ ffmpeg6.exe │ ─────→  │ ffmpeg.exe   │
│          │         │   (Proxy)   │         │  (Real AMF)  │
└──────────┘         └─────────────┘         └──────────────┘
   libx264              converts                  h264_amf
   (CPU)                                          (GPU)
```

---

## ✨ Features

- AMD AMF encoders: h264_amf (H.264/AVC), hevc_amf (H.265/HEVC), av1_amf (AV1).
- Transparent conversion: libx264 → h264_amf without changing your vMix profile.
- Preset mapping for low latency and balanced quality/speed trade-offs.
- FDK-AAC integrated and statically linked for high-quality AAC audio.
- Static Windows binaries (no external DLLs), cross-compiled from Linux/WSL2.
- Full logs: depure.log (build) and ffmpeg_proxy.log (runtime) for quick diagnostics.

---

## 🚀 Quick Start

### Build (Linux/WSL2)
- Ensure Ubuntu 20.04+ or WSL2 with basic build tools.
- Run the provided build script (vmixproxy.sh).
- Resulting artifacts will be in output/.

### Deploy (Windows)
Copy these files from output/ to %ProgramFiles(x86)%\vMix\streaming directory on Windows:
- ffmpeg6.exe — proxy (configure vMix to use this)
- ffmpeg.exe — real FFmpeg with AMF and FDK-AAC
- ffprobe.exe — analysis utility

### Configure in vMix
- Settings → Encoders → External → point to ffmpeg6.exe.
- Keep your normal H.264/libx264 settings (bitrate, preset, etc.).
- The proxy converts to AMF automatically and uses the AMD GPU.

---

## 💻 Usage Examples

vMix external FFmpeg (conceptually):
- Your profile: H.264 (libx264), veryfast, 6000 kbps
- Proxy conversion: h264_amf, usage=lowlatency, quality=speed, 6000 kbps

---

## 🔄 How it Works

The proxy inspects and transforms common H.264 options into AMF equivalents:

| Original (x264)      | Converted (AMF)                        | Purpose          |
|----------------------|----------------------------------------|------------------|
| -codec:v libx264     | -codec:v h264_amf                      | Use AMD GPU      |
| -preset ultrafast    | -usage speed -quality speed            | Max speed        |
| -preset veryfast     | -usage lowlatency -quality speed       | Low latency      |
| -preset fast         | -usage lowlatency -quality balanced    | Balanced         |
| -preset medium       | -usage transcoding -quality balanced   | Quality/balance  |
| -preset slow/slow+   | -usage transcoding -quality quality    | Highest quality  |
| -crf N               | -b:v (heuristic mapping)               | Stable bitrate   |
| -tune zerolatency    | (removed)                              | Not needed in AMF|

Notes:
- CRF-to-bitrate mapping is heuristic to preserve approximate visual quality; adjust -b:v to your needs.
- Flags without an AMF equivalent are safely ignored to avoid errors.

---

## 📊 Performance (Typical)

- CPU usage drops from ~100% to ~10–20% by moving H.264 encoding to the AMD GPU.
- GPU usage typically 60–80% during live encoding (depends on card and settings).
- Lower CPU temperature and improved system responsiveness under streaming load.
- Low-latency path enabled via AMF usage lowlatency where applicable.

---

## ✅ Requirements

Runtime (Windows):
- Windows 10/11
- AMD GPU RX 400 series or newer
- AMD Adrenalin drivers 22.10.1 or newer

Build (Linux/WSL2):
- Ubuntu 20.04+ or WSL2
- Standard build tools (installed by script)
- vmixproxy.sh script

---

## 🐛 Troubleshooting

Common checks:
- Both ffmpeg6.exe (proxy) and ffmpeg.exe (real) must be in the same directory.
- Verify AMF encoders are present: ffmpeg.exe -encoders | findstr amf
- If vMix fails to start FFmpeg, open ffmpeg_proxy.log to see the rewritten command line.

FDK-AAC dependency:
- The build script includes explicit include/lib paths for FDK-AAC, resolving "libfdk_aac not found" on clean systems.

Quality tuning:
- If the result is too soft at your CRF, increase -b:v or switch to a higher-quality preset mapping (e.g., medium → transcoding+balanced, slow → transcoding+quality).

---

## 📝 Changelog

### v0.1b — DirectShow Buffer Fix & Optimization
**Release Date:** January 9, 2026

**DirectShow Buffer Optimization**
- Solves frequent "real-time buffer [vMix Video YV12] [video input] too full or near too full" warnings causing frame drops during live streams.
- Automatic override of the `-rtbufsize` parameter from vMix's default 128M to 1024M (1 GiB), providing sufficient buffer headroom for sustained 1080p capture without frame loss.

**Production Validation Results**
- Tested on Asus TUF FA617NT (Ryzen 7 7735HS, 64GB RAM & Radeon RX 7700S).
- Input: 1920x1080 @ 29.97 fps via vMix Video YV12.
- Target bitrate: 6000 kbps (configured in vMix).
- Actual delivery: 5.6–5.8 Mbps (stable throughout the stream).
- Encoding speed: 1.02–1.05x realtime.
- Buffer warnings: zero (complete elimination).
- Frame drops: zero after initial startup buffering.

**Enhanced Logging**
- Per-run logs capture both the original and rewritten commands with clear formatting.
- Example log output:
  ```
  ==== vmixproxy per-run log ====
  Original command: "C:\Program Files (x86)\vMix\streaming\ffmpeg6.exe" -report -rtbufsize 128M ...
  Final command: "C:\Program Files (x86)\vMix\streaming\ffmpeg.exe" -report -rtbufsize 1024M -c:v h264_amf ...
  ```

**Core Features (unchanged from v0.1a)**
- Drop-in proxy that replaces libx264 with h264_amf while preserving all vMix encoding parameters (bitrate, GOP, profile, level, audio).
- Maps x264 presets (ultrafast → veryslow) to AMF `-usage` / `-quality` pairs.
- Launches the real ffmpeg.exe via the Win32 process API and transparently forwards its exit code back to vMix.
- Absolute path execution: fixed path to `C:\Program Files (x86)\vMix\streaming\ffmpeg.exe`.

**Technical Notes**
- The 1024M buffer allocation is memory-only (system RAM) and does not consume VRAM (~1.6% of 64GB total).
- Bitrate delivery remains within approximately 3–5% of target (typical VBR-like behavior under CBR constraints).
- **Breaking Changes:** None. Full backward compatibility with v0.1a deployments.

### v0.1a — First Release
- Transparent proxy for libx264 → h264_amf (vMix-friendly).
- FFmpeg static build with AMF and FDK-AAC integration.
- Definitive fix for “libfdk_aac not found” via explicit include/lib paths.
- Robust preset mapping and safe removal of incompatible flags.
- Detailed build and runtime logs for easy debugging.

### Previous (internal) milestones:
- v46.1: Added FDK-AAC artifact/header verification and clearer errors.
- v46.0: Initial working proxy + AMF pipeline validation.

---

## 🦜🎥 Support

This tool is the culmination of many hours of work, developed as part of the RDI (Research, Development, and Innovation) initiatives at Ponto de Cultura Amazônia Audiovisual (a non-profit entity).

Based in Manaus, at the heart of the Amazon Rainforest, we are a non-profit organization committed to advancing the audiovisual sector and creative economy. We provide artists and cultural producers with an environment to realize their projects by offering technical qualification, promoting their works, and fostering the sector's long-term sustainability.

If this project helped you and you want to support future development, consider buying a coffee:

👉 https://buymeacoffee.com/amazoniaaudiovisual

Official Website: https://www.amazoniaaudiovisual.com.br

YouTube Channel: https://www.youtube.com/@amazoniaaudiovisual9838

Our Latest Production "Grana Preta": https://www.youtube.com/watch?v=JWzKAfYejc8

---

## 🤝 Contributing

- Open issues and feature requests.
- Share logs (depure.log, ffmpeg_proxy.log) for bug reports.
- Pull requests welcome (build steps and proxy improvements).

---

## 💙 Acknowledgements

- FFmpeg community
- AMD GPUOpen (AMF)
- Early testers who validated the vMix workflow

---

## 📄 License

- Build scripts: MIT
- FFmpeg: GPL v3 / LGPL v2.1
- AMD AMF SDK: AMD Software License

---

## 📫 Contact

- Issues: open a ticket with details and logs.
- Discussions: propose improvements and vote on roadmap items.

💚💛 Made for streamers and integrators who want AMD GPU acceleration without changing their existing vMix setups.

