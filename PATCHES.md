# Yeti patches to GPUI

`hamropatro/gpui` is `Glass-HQ/gpui` plus the patches below, kept on the `yeti` branch for the
Yeti Chat desktop app (`hamropatro/circle-chat-desktop-rust`, which pins a revision of this fork in
its `Cargo.toml`). Changed code is marked with `yeti:` comments where practical.

When updating from upstream, re-apply each patch, keep its comment, and update this list. To see
everything: `git diff 650aab5 yeti` (650aab5 is the upstream commit the fork started from).

| # | Files | Change | Why |
|---|---|---|---|
| 1 | `elements/surface.rs`, `scene.rs`, `window.rs`, `gpui_metal/renderer.rs`, `gpui_metal/shaders.metal` | Surfaces (CoreVideo `CVPixelBuffer`) accept corner radii: `PaintSurface` carries mask bounds and radii, the surface fragment shaders apply a quad SDF mask, and the element clips to its own bounds (`Window::paint_surface_rounded`) | Video drawn straight from AVPlayer keeps the card's rounded corners, even when cover-fit overflows |
| 2 | `gpui_metal/atlas.rs`, `gpui_wgpu/wgpu_atlas.rs`, `gpui_windows/directx_atlas.rs` | Removing an atlas tile deallocates it (`AtlasAllocator` instead of the bucketed one), 2048² textures | Streaming images (video posters, avatars) reuse atlas space instead of leaking textures |
| 3 | `app.rs` | Image asset cache bounded to 80 entries (LRU), evicted images release their atlas tiles (deferred when a window is checked out) | Long feeds don't grow RAM without bound |
| 4 | `elements/img.rs` | Decoded images capped at 2400 px on the long side | Huge originals cost tens of MB each |
| 5 | `text_system/line_wrapper.rs` | Devanagari (U+0900–097F, A8E0–A8FF) and ZWJ/ZWNJ are word characters | Nepali lines no longer break between a letter and its vowel sign |
| 6 | `taffy.rs`, `Cargo.toml`, `build.rs`s, `layer_shell.rs` | Build fixes (typed float literals, ashpd 0.13, Windows paths, `LayerShellNotSupportedError`) | Builds on current toolchains and Windows |
| 7 | `window.rs` | `Window::render_offscreen`, `draw_frame_now`, `needs_draw` | The app's scripted UI driver: screenshots of a parked or covered window, one frame per benchmark step |
| 8 | `gpui_metal/renderer.rs` | Path intermediate and MSAA textures created on the first frame with paths, dropped after 1 s without any and on resize | A window-sized texture (~20 MB at 2×) only while paths are drawn (from quiet.md) |
| 9 | `elements/div.rs` | On a trackpad, a one-axis scroll container only takes gestures leaning its way | A vertical swipe's sideways drift no longer slides horizontal strips (from quiet.md; mouse wheels unchanged) |
| 10 | `platform.rs`, `window.rs`, `gpui_macos/window.rs` | `PlatformWindow::note_frame_activity` / `frame_waker`: the macOS display link stops after 3 idle frames and restarts when the window needs a frame; parked over 30 s it is recreated | Idle windows cost no per-refresh wakeups (from zeron): idle chat 86 → 26 wakeups/s |
| 11 | `elements/img.rs` | Decoded images turned upright by their EXIF orientation | Phone photos stored sideways show upright |
| 12 | `gpui_metal/renderer.rs` | Two drawables per layer instead of three | One window-sized buffer less (~15–30 MB at 2×) (from zeron) |
| 13 | `window.rs`, `platform.rs`, `app.rs`, `gpui_macos/platform.rs` | Frame-rate caps apply to every redraw (not only forced frames): 30 fps inactive or in Low Power Mode (`App::low_power_mode`), 60 fps when hot; 2 ms slack for refresh jitter | Background video and animations cost less; a 60 fps cap on 120 Hz no longer drops to 40 (from Zed) |
| 14 | `debug_overlay.rs` (from Zed, Apache-2.0), `window.rs` | Frame-time overlay painted straight into the scene; `Window::{set_,cycle_}debug_frame_overlay_mode`, `reset_debug_frame_overlay_stats` | Live frame times without extra frames |
| 15 | `text_system.rs`, `text_system/line_layout.rs`, `text_system/font_features.rs`, `text_system/line.rs`, `elements/text.rs`, `window.rs` | Text allocation trims from Zed: a line-layout cache hit no longer copies the text (`layout_line_str`), no 32-run reservation per line, one shared empty `FontFeatures`, no background pass for lines without backgrounds, the line wrapper taken only when truncating, the cheap subpixel check first | Less allocation per line while scrolling text-heavy views |
| 16 | `elements/list.rs` | `ListState::set_smooth_follow`: while following the tail, new content is painted a little low and a critically damped spring brings it into place (~250 ms) | New chat messages glide in instead of jumping (after zeron's stick-to-bottom spring) |
| 17 | `elements/list.rs` | `ListState::set_rubber_band`: trackpad scrolling past either end stretches the content (40% resistance, up to 120 px) and it springs back once input stops | macOS-style overscroll bounce (after zeron's overscroll) |
