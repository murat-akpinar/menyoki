# menyoki 1.8.0 — Wayland/Hyprland test report

Binary: `master` @ `1183d7e` (v1.8.0), release build
Compositor: Hyprland 0.56.2
Output: eDP-1, 1920x1080@60, scale 1, transform 0 (single monitor)
Date: 2026-09-12

Reference tool for pixel comparisons: `grim` + `magick compare -metric RMSE`
(0 = pixel-identical).

## Summary

38 checks run, 33 pass, 5 issues (1 hang, 4 minor). Core paths — output
capture, window capture, recording, every file format and every non-window
subcommand — work correctly on Hyprland.

The strongest results:

* `capture --root` is **pixel-identical to `grim`** (RMSE 0) — channel order,
  y-flip and stride handling are all correct.
* `capture --focus` of an **occluded window is identical** (RMSE 0) to the
  capture of the same window when nothing covers it, while the screen region
  at that place differs (RMSE 0.105). The README claim that a window is
  captured "on its own, without whatever happens to be drawn on top of it"
  holds.

## Passing

| # | Test | Result |
|---|------|--------|
| 1 | Backend picked from `WAYLAND_DISPLAY` | Wayland backend used, `Output name -> "eDP-1"` |
| 2 | `capture --root` | 1920x1080 PNG, **RMSE 0 vs grim** |
| 3 | `capture --root --size 800x600` | 800x600 |
| 4 | `capture --root --padding 100:200:300:400` | 1320x680 (= 1920-200-400 x 1080-100-300) |
| 5 | `capture --root --monitor 1` | 1920x1080 |
| 6 | `capture --root --monitor 2` (invalid) | `Invalid monitor number: 2 (found 1 outputs)`, exit 1 |
| 7 | `capture --root --with-alpha` | 1920x1080 RGBA |
| 8 | `capture --root --countdown 1` | counts down, then captures |
| 9 | `capture --root --mouse` | warns `Selecting a window with the mouse is not supported on Wayland.` |
| 10 | `capture --parent` with no focused window | warns about `--parent`, then `No focused window found to capture.`, exit 1 |
| 11 | `capture` (default = focused window) | 800x600, window title reported |
| 12 | `capture --focus` | 800x600 |
| 13 | `capture --focus --size 400x300` | 400x300 |
| 14 | `capture --focus --padding 10:20:30:40` | 740x560 |
| 15 | `capture --focus --with-alpha` | 800x600 RGBA |
| 16 | `capture --focus` with another window raised on top | **RMSE 0** vs the unoccluded capture |
| 17 | `capture --focus` on an empty workspace | `No focused window found to capture.`, exit 1 |
| 18 | 10 consecutive `capture --focus` runs | 10/10 ok, no toplevel-state race observed |
| 19 | `record --root --duration 2` (gif) | 40 frames @20 FPS, 1920x1080 |
| 20 | `record --focus --duration 2` (gif) | 40 frames, 800x600 |
| 21 | `record --focus --duration 2` (apng) | 40 frames (`acTL` + 40 `fcTL`; ImageMagick can't read APNG, ffprobe can) |
| 22 | `record` warns about action keys | `Action keys are not supported on Wayland, use --duration or press Ctrl-C` |
| 23 | Ctrl-C during recording | GIF saved with 41 frames, exit 0 |
| 24 | Recording with no `--duration`, stopped by Ctrl-C | 30 frames saved, exit 0 |
| 25 | `record --root "sleep 2"` (COMMAND mode, async recorder) | 10 frames @5 FPS, exit 0 |
| 26 | `capture --root "sleep 1"` (COMMAND mode) | PNG saved, exit 0 |
| 27 | Capture formats: png, jpg, webp, bmp, tiff, tga, ff, exr | all written, all 320x240 |
| 28 | `pnm --format pixmap` | PPM 320x240 |
| 29 | `ico` | 256x240 — ICO size cap applied by design (`set_icon_size`) |
| 30 | `gif --gifski` | 40 frames on moving content (1 frame on a static screen is gifski's duplicate-frame collapsing, not a bug) |
| 31 | `split` | 20 PNG frames out of a 20-frame GIF |
| 32 | `make 1.png 2.png save out.gif` | 3-frame GIF |
| 33 | `analyze` (+ `save report.txt`) | report printed and saved (534 B) |
| 34 | `edit --resize / --grayscale / --convert` | 100x100, grayscale, jpg — all ok |
| 35 | `view` | exit 0 |

## Issues

### 1. Capture can hang forever when the window disappears mid-copy (highest severity)

Observed once: `capture --focus` run right after the focused window was
closed selected the dying window (it printed `Window name -> "menyoki-race"`),
then blocked in `ppoll` on the Wayland socket for 8+ minutes until killed.

Cause (src/wayland/display.rs, `copy_frame`):

```rust
while state.frame_status == FrameStatus::Pending {
    queue.blocking_dispatch(state)?;
}
```

There is no deadline. If the compositor never sends `Ready` or `Failed` for a
frame whose toplevel was destroyed mid-request, menyoki waits forever. The
same loop is on the `record` path, so a recording can hang the same way.

Not reproducible on demand: 20 targeted attempts (close / `kill -9` the client
at delays of 20-300 ms while a capture was in flight) all exited cleanly with
`The compositor did not offer a buffer`. So it is a narrow race, but the wait
is unbounded, which is what makes it a hang rather than an error.

Suggested fix: bound the wait (deadline around the dispatch loop) and fail
with the existing error message instead of blocking.

### 2. Recorded frames are thrown away if the window closes mid-recording

`record --focus`, window closed while recording:

```
ERROR menyoki::wayland::window] The compositor did not offer a buffer
ERROR menyoki] Frame error: `Failed to get image`
exit=1
```

No file is written, and every frame captured up to that point is lost. Ending
the recording and encoding what was already captured would be friendlier — the
frames are already in memory.

### 3. Resizing a window mid-recording produces a smeared band

The capture area is fixed when recording starts. After shrinking the window
from 800x600 to 500x400, recording continued at 800x600 and the region past
the new window edge became a repeated edge column (measured: standard
deviation 0 across columns 500-799 of the last frame). `get_pixels` clamps
out-of-range coordinates instead of reporting a size change.

Not data loss, and the fixed area is a deliberate design choice, but the
output is silently wrong for the rest of the recording.

### 4. `--select` is silently ignored

The README says `--select` and `--mouse` are not available on Wayland.
`--mouse` warns; `--select` produces no message at all and simply captures the
whole output / focused window. One extra `warn!` in `get_window` would make
the two consistent.

### 5. Ctrl-C during the countdown reports an error

Interrupting during the countdown (before any frame is captured) exits with
`Frame error: No frames found to save` and exit code 1, instead of a plain
"cancelled". Cosmetic.

### 6. (Not Wayland) `MENYOKI_WINDOW_SYSTEM=x11` with no X display panics

```
Could not connect to a X display
note: run with `RUST_BACKTRACE=1` ...
timeout: the monitored command dumped core
```

This is the X11 backend, not the new code, but the Wayland docs point users at
that variable for XWayland windows, so it is easy to hit from a Wayland
session with no `DISPLAY`.

## Not covered

No hardware for these here — worth checking before claiming they work:

* **Multi-monitor**: only one output exists on this machine, so `--monitor 2+`,
  the "N outputs found, capturing the first one" warning and per-output
  geometry were only exercised in their error paths.
* **Fractional scaling** (`scale != 1`) and **rotated outputs**
  (`transform != 0`): output geometry comes from `wl_output.mode`, which is the
  physical mode size. On a rotated output the screencopy buffer has width and
  height swapped; `get_pixels` clamps instead of erroring, so the crop would be
  wrong rather than refused. Untested.
* **Non-Hyprland wlroots compositors** (Sway, river): `--root` should work via
  wlr-screencopy, `--focus` should print the "this compositor does not allow
  capturing an individual window" error. Untested.

## Fix plan (one PR)

| # | Issue | Where | Change |
|---|-------|-------|--------|
| 1 | Unbounded wait can hang | `src/wayland/display.rs:464` (`copy_frame`) | Bound the `blocking_dispatch` loop with a deadline; on timeout return the existing `Err` instead of blocking. `Connection::prepare_read` + `poll` with a timeout, or a wall-clock check per iteration. |
| 2 | Frames lost when the window closes mid-recording | `src/record/mod.rs:138` (`record_sync`), `:163` (`record_async`) | On `get_image()` failure, break out with the frames collected so far instead of propagating `FrameError`; warn that the recording ended early. Keep the error when no frames exist. |
| 3 | Smeared band after a mid-recording resize | `src/wayland/display.rs:565` (`get_pixels`), area fixed in `src/wayland/mod.rs:52` | Detect that the frame buffer no longer covers `area` (`info.width`/`info.height` shrank) and stop the recording with a clear message rather than clamping silently. |
| 4 | `--select` silently ignored | `src/wayland/mod.rs:54` | Warn for `flag.select` next to the existing `flag.mouse` warning, matching the README. |
| 5 | Ctrl-C during the countdown reports an error | `src/record/mod.rs:121-124` | The handler is installed before `show_countdown()`; if the interrupt arrives before the first frame, exit cleanly ("recording cancelled") instead of `No frames found to save`. |
| 6 | X11 backend panics with no `DISPLAY` | X11 backend (`src/x11/display.rs`) | Turn the `Could not connect to a X display` panic into a normal error exit; it is reachable from a Wayland session via `MENYOKI_WINDOW_SYSTEM=x11`. |

Issues 1-5 are in the Wayland backend; 6 is pre-existing X11 behaviour and can
be split out if upstream prefers a narrower PR.
