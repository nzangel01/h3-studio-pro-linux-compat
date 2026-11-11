# H3 Studio Pro — Linux / non-NVIDIA Compatibility Report

Not an install of H3 Studio Pro itself, and contains none of its source. This is a
compatibility report from testing whether the AI engine behind H3 Studio Pro (a
Windows app) runs on Linux and on non-NVIDIA GPUs, written up for the app's own
developer.

## TL;DR

**Yes — no code changes needed.** The engine runs cleanly on both an Intel Arc
B580 (XPU backend) and an AMD RX 7900 XTX (ROCm backend), as long as 2 node
packs that are hard-locked to NVIDIA are excluded.

## What's actually inside H3 Studio Pro

The Windows installer's `engine/` folder is:
- `engine/core` — a standard [ComfyUI](https://github.com/comfyanonymous/ComfyUI)
  checkout (same `main.py`, `nodes.py`, `server.py`, `comfyui-frontend-package`
  as upstream), plus a `custom_nodes/` folder.
- `engine/rt` — a bundled Windows Python runtime (not needed on Linux; use a
  normal Linux Python/venv instead).
- The Windows `.exe` itself is an Electron shell that just points at the
  ComfyUI web UI — not required at all, since ComfyUI already ships its own
  web UI.
- First run downloads models (`z_image_turbo`, `qwen_3_4b` text encoder, both
  from the public Comfy-Org HuggingFace repos) via a signed manifest.

`custom_nodes/` is a mix of well-known public ComfyUI nodes (Manager,
KJNodes, rgthree, VideoHelperSuite, Crystools, Frame-Interpolation, GIMM-VFI,
easy-use, custom-scripts, layerstyle, etc.) plus the product's own nodes
(`ComfyUI-H3-Motion-Context-MultiRef`, `ComfyUI-H3-Multishot`,
`Comfyui_Minimax_h3_latent_Upscaler`, `minimax-h3-audio-t8`). All of these are
pure Python — static scan found no compiled `.pyd`/`.dll`/`.so` files in any of
them.

## The 2 exceptions (NVIDIA-locked, exclude on AMD/Intel)

| Node | Why it's NVIDIA-only |
|---|---|
| `ComfyUI-RH-DLSS5` | Wraps NVIDIA DLSS 5 noise reduction. Ships a prebuilt Windows `.dll`/`.exe` only (source `.cpp` + a Linux build script exist but no Linux binary is built). Optional feature (noise reduction), not core to generation. |
| `comfyui_nvidia_rtx_nodes` | Named and scoped to NVIDIA RTX specifically. |

Everything else installed and loaded without modification.

## What was tested

| Machine | GPU | Backend | Result |
|---|---|---|---|
| silvia | Intel Arc B580 | `torch` XPU build | Clean boot, 0 import errors, `Device: xpu:0 Intel(R) Arc(TM) B580 Graphics` |
| yuki | AMD RX 7900 XTX | `torch` ROCm 6.3 build | Clean boot, 0 import errors, `Device: cuda:0 Radeon RX 7900 XTX : native` (ROCm reuses the `cuda` device namespace — expected) |

Both runs logged the H3-specific nodes initializing normally, e.g.:

```
[H3AVBank] extra_conds merge patch active (refs + keyframes compose; Motion-Context-compatible)
[H3RemoteEncode] serving POST /h3multishot/encode (clips load on first request)
[INFO] [H3-Multishot] LTX rational upsampler: blur kernel follows the input device (partial-load safe)
```

That last line is from the product's own code — it already picks its compute
device dynamically rather than hardcoding CUDA, which lines up with what we
saw: no patch was needed anywhere in the H3-specific nodes.

One non-fatal warning on both machines, expected on non-NVIDIA hardware:

```
[Crystools ERROR] Could not init pynvml (NVIDIA). NVML Shared Library Not Found
```

Crystools is a GPU-monitoring node; it just can't query NVIDIA-specific
counters on non-NVIDIA cards. It doesn't block anything else from running.

## Known unknown

`minimax-h3-audio-t8/h3_t8/trt_vae_backend.py` references TensorRT (NVIDIA-only).
The default workflow we tested doesn't appear to call this path, so it never
got exercised. Whether there's a non-TensorRT fallback for whatever feature
uses it is untested — flagging it rather than claiming it's fine.

## Method (so this is reproducible)

1. Copy `engine/core` out of the Windows install.
2. Static-scan `custom_nodes/` for hardcoded `cuda:0` / `.cuda()` strings and
   for vendor-locked imports (`cupy`, `tensorrt`, `pynvml`, etc.) — mostly
   boilerplate GPU-detection code with graceful fallbacks, not hard blocks.
3. Stand up vanilla ComfyUI with the matching GPU backend (`torch` XPU build
   for Intel, ROCm build for AMD).
4. Drop in every `custom_nodes/*` folder except the 2 NVIDIA-locked ones above.
5. Install each node's own `requirements.txt`.
6. Restart, check logs for `IMPORT FAILED` / `Traceback` / `ModuleNotFoundError`.

No proprietary H3 code is included in this repo — just the findings above, for
the developer to act on in their own copy.
