# Recording & Capture (the render layer)

The render/capture layer owns the D3D11 device, the in-app MP4 recorder, and the
optional supersampling (SSAA) resolve pass. It lives in `src/app/render/` and is
aggregated by `src/app/render_capture.inc`, which `main.cpp` `#include`s once:

```
render_capture.inc
 ├─ render/device_d3d11.inc   // device / swapchain / RTV lifecycle (+ <d3dcompiler.h>, ImGui WndProc extern)
 ├─ render/recorder_mp4.inc   // F9 screen recorder -> ffmpeg -> MP4
 └─ render/ssaa_resolve.inc   // supersample downsample pass
```

`device_d3d11.inc` is included first because it pulls in `<d3dcompiler.h>`, which
`ssaa_resolve.inc` needs for `D3DCompile`. All three operate on file-scope globals
declared in `main.cpp` (`g_pd3dDevice`, `g_pSwapChain`, the `g_rec_*` recorder block,
the `g_ssaa*`/`g_resolve*` SSAA block).

---

## 1. Device / swapchain (`device_d3d11.inc`)

| Function | Role |
|---|---|
| `CreateDeviceD3D(hWnd)` | Creates the swapchain + device. Tries `D3D_DRIVER_TYPE_HARDWARE`; on `DXGI_ERROR_UNSUPPORTED` falls back to the **WARP** software rasterizer so the overlay still runs on machines without a usable GPU. Then `CreateRenderTarget()`. |
| `CleanupDeviceD3D()` | Releases RTV, swapchain, context, device. |
| `CreateRenderTarget()` / `CleanupRenderTarget()` | (Re)build the backbuffer RTV — called on init and on window resize. |

The swapchain is a classic `DXGI_SWAP_EFFECT_DISCARD`, 2-buffer, windowed chain at the
client size; `Present(1, 0)` (vsync) is called from the main loop.

---

## 2. MP4 recorder (`recorder_mp4.inc`)

Toggled with **F9**. It records the **rendered backbuffer**, so — unlike OBS/Snipping
Tool — it is unaffected by the `WDA_EXCLUDEFROMCAPTURE` display affinity the overlay
normally sets. Pipeline:

```
backbuffer ──CopyResource──▶ staging tex ──Map/BGRA pack──▶ g_rec_buf ──WriteFile──▶ ffmpeg stdin (pipe)
                                                                                    └─▶ H.264 MP4
```

| Function | Role |
|---|---|
| `RecorderStart(W,H)` | Opens a pipe, spawns a child **ffmpeg** with `CreateProcessA` (no window), wiring our pipe to ffmpeg's stdin. ffmpeg encodes **2560×1440 @ 60 fps**, letterboxed to preserve aspect, **10 Mbit/s** capped H.264 (`-b:v 10M -maxrate 10M`), `+faststart`. Allocates the BGRA frame buffer. |
| `RecorderFrame(dt)` | Called once per frame before `Present`. Copies the backbuffer to a staging texture, packs BGRA top-down into `g_rec_buf`, and `WriteFile`s it to the pipe — **gated to ~30 fps** via an accumulator. Auto-stops if ffmpeg closes the pipe. |
| `RecorderStop()` | Closes the pipe (EOF → ffmpeg finalizes the MP4), waits for the process, frees buffers. |

The loop also **auto-stops after 2 minutes**.

**ffmpeg location:** `g_ffmpeg_exe` (default `C:/ffmpeg/ffmpeg.exe`); if missing it
falls back to `ffmpeg.exe` on `PATH`. If ffmpeg can't be launched at all, recording
simply no-ops. **Output:** `g_rec_path` (default `%USERPROFILE%/glass_recording.mp4`).
ffmpeg's stdout/stderr go to `%TEMP%/glass_ffmpeg_log.txt` (a GUI app has no console).

> To record for OBS instead, toggle **screenshot mode** (clears `WDA_EXCLUDEFROMCAPTURE`
> so external capture tools can see the overlay).

---

## 3. SSAA resolve (`ssaa_resolve.inc`)

An optional supersampling path (Tier-3 quality, **off by default** — gated off by
`ssaaOn = false`; `ssaaScale` itself defaults to `2.0`, so flipping `ssaaOn` on renders
at 2×). When enabled, the scene renders into an oversized offscreen target and is
downsampled to the backbuffer with a fullscreen-triangle shader.

| Function | Role |
|---|---|
| `InitResolve()` | Compiles `kResolveHLSL` (a fullscreen-triangle VS + a single-tap bilinear PS), and creates the linear-clamp sampler + opaque blend state. |
| `EnsureSSAA(w,h)` | Lazily (re)allocates the supersample RT/SRV at the requested size. |
| `ResolveSSAA()` | Binds the SSAA SRV + resolve shaders and draws 3 verts — a bilinear tap = a 2×2 box average at 2× scale. |
| `CleanupSSAA()` | Releases the SSAA textures/views. |

`kResolveHLSL` is a tiny inline HLSL string (it does **not** go through the big
`shaders.h` glass shader); it only resolves the oversampled image down to native res.

---

## Where the knobs live

- ffmpeg path / output path: `g_ffmpeg_exe`, `g_rec_path` (top of `main.cpp`).
- SSAA on/scale: the `ssaaOn` / `ssaaScale` UI vars (Customize → Graphics Quality),
  pushed into `g_ssaaScale` each frame.
- The recorder UI (Record/Stop button, elapsed time, OBS toggle) is in
  `app/pages/account.inc` (the Account Settings page, RECORDING panel).
