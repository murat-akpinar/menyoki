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

## Fix roadmap

Branch: `fix/wayland-issues`. One finding per commit, each one built, unit
tested (`cargo test`) and re-checked against the live compositor before the
next one is started. Every entry below is filled in with its cause and the
fix that was applied as it lands.

Progress:

- [x] 1. Capture can hang forever when the window disappears mid-copy
- [x] 2. Recorded frames are thrown away if the window closes mid-recording
- [x] 3. Resizing a window mid-recording produces a smeared band
- [ ] 4. `--select` is silently ignored
- [ ] 5. Ctrl-C during the countdown reports an error
- [ ] 6. `MENYOKI_WINDOW_SYSTEM=x11` with no X display panics

2 is fixed before 3 on purpose: 3 turns a silent wrong crop into an error,
and 2 is what keeps that error from throwing the recording away.

### 1. Capture can hang forever when the window disappears mid-copy

**Status:** fixed

**Finding:** a `capture --focus` of a window that was closing blocked in
`ppoll` on the Wayland socket for 8+ minutes instead of failing.

**Cause:** `copy_frame` (`src/wayland/display.rs`) waited for `Ready`/`Failed`
in `while state.frame_status == FrameStatus::Pending { queue.blocking_dispatch(state)?; }`.
`blocking_dispatch` blocks until the compositor sends *something*, and the
loop has no deadline, so a frame that is never answered for — the toplevel
was destroyed while the copy was in flight — blocks forever. `record` uses
the same loop, so a recording can hang the same way.

**Fix:** the wait is bounded by a 5 second deadline (`FRAME_TIMEOUT`). The
loop now uses `queue.roundtrip()` instead of `blocking_dispatch()`: a
roundtrip returns as soon as the compositor answers a `wl_display.sync`,
which makes the deadline check reachable, and the frame events that arrive in
the meantime are dispatched exactly as before. A 1 ms sleep
(`FRAME_CHECK_INTERVAL`) between the checks keeps an unanswered frame from
spinning the CPU. On timeout the existing error path is taken, so menyoki
exits 1 with `The compositor did not send the frame` instead of blocking.

Polling the socket with `prepare_read` + `poll` would avoid the 1 ms
granularity, but it needs a direct `rustix`/`libc` dependency; that is noted
as the upgrade path in the code.

**Verification:**

* `capture --root` still **RMSE 0 vs grim**, 0.06 s end to end (no added
  latency from the sleep).
* `record --root --duration 2` still 40 frames, 1920x1080.
* 20 close-during-capture attempts (alternating `closewindow` and `kill -9`,
  0-140 ms after the capture starts): 20 exits, 0 hangs.
* Deadline branch exercised directly by building with `FRAME_TIMEOUT` set to
  0 s: the capture fails in 3 ms with `The compositor did not send the frame`
  and exit 1, where the old code would have blocked.
* `cargo fmt --check`, `cargo clippy --tests -- -D warnings`, `cargo test`
  (36/36) all pass.

### 2. Recorded frames are thrown away if the window closes mid-recording

**Status:** fixed

**Finding:** closing the window during `record --focus` exits 1 with
`Frame error: Failed to get image` and writes no file; every frame captured
until then is lost.

**Cause:** three places dropped the frames on the floor. `record_sync` turned
a `None` frame into `AppError::FrameError` and returned, discarding the
frames it already held. `record_async` panicked through
`expect("Failed to get the image")` instead. And `RecordResult::get` only
joined the recording thread when its stop message was delivered — a thread
that had already finished on its own has dropped the receiver, so the send
failed, `get` returned `None`, and the caller substituted an empty frame
list. Any one of the three was enough to lose the recording.

**Fix:** a failed frame now ends the recording instead of failing it: both
loops warn `The recording ended early.` and break, keeping what they have.
`record_sync` still returns the original error when the *first* frame fails,
since there is nothing to save then. `RecordResult::get` always joins and
returns `thread::Result<T>` — the `Option` only encoded the case that lost
the frames, so it is gone, along with the `None` arm in `App::record`.

**Verification:**

* `record --focus --duration 10`, window killed ~3 s in: **60 frames saved,
  exit 0** (was: no file, exit 1).
* Same run with the window killed during the countdown, before any frame:
  still `Frame error: Failed to get image`, exit 1, no file — as intended.
* Async path, `record --focus --countdown 0 'sleep 8'` with the window killed
  ~3 s in: **59 frames saved, exit 0** (was: panic in the recording thread,
  frames replaced by an empty list).
* Regression, async path unharmed: `record --root --countdown 0 'sleep 2'`
  still 40 frames, 1920x1080, exit 0.
* `cargo fmt --check`, `cargo clippy --tests -- -D warnings`, `cargo test`
  (36/36) all pass.

Noticed while testing, not a regression and not fixed here: a COMMAND that
finishes before the countdown does (`record --root 'sleep 2'` with the
default 3 s countdown) records nothing and exits with
`No frames found to save`. It behaves the same way before this branch.

### 3. Resizing a window mid-recording produces a smeared band

**Status:** fixed

**Finding:** after shrinking an 800x600 window to 500x400 mid-recording, the
region past the new window edge became a repeated edge column for the rest of
the recording (standard deviation 0 across columns 500-799).

**Cause:** the capture area is fixed when the recording starts, but the
frames the compositor hands over shrink with the window. `get_pixels`
clamped every out-of-range row and column to the last pixel of the buffer
(`.min(info.width - 1)`, `.min(info.height - 1)`), which turns "this frame is
too small" into "repeat the edge pixel", silently, for every remaining frame.

**Fix:** `copy_frame` now rejects a frame that does not cover the requested
area, with
`The capture area (800x600 at 0,0) does not fit in the frame (759x573)`.
Combined with fix 2, the recording ends there and keeps its frames instead of
filling them with smear. The two `.min()` clamps in `get_pixels` are gone —
the guard makes every coordinate in range by construction, so clamping could
only hide the next bug of this kind.

This also changes what a rotated output does: `wl_output.mode` reports the
untransformed mode while the screencopy buffer is transformed, so a
`transform != 0` output now fails with the message above instead of writing a
wrongly cropped image. Untested here, no rotated output on this machine.

**Verification:** 800x600 window recorded with
`record --focus --countdown 0 --duration 10`, resized to 500x400 three
seconds in, running `base64 /dev/urandom` so the content varies across the
full width. A smeared band is constant along every row, so the band past
x=500 of the last frame is compared against its own row averages:

| | frames | band vs row averages | |
|---|---|---|---|
| before (fixes 1-2 only) | 200 (full 10 s) | **RMSE 0** | smeared |
| after | 59 (stops at the resize) | RMSE 0.172 | real content |

* `capture --root` still RMSE 0 vs grim, `capture --focus` unaffected: an
  area that fits is never rejected.
* New unit test `wayland::display::tests::test_get_pixels` covers the pixel
  math the clamps were part of — channel order and the `y_invert` flip.
* `cargo fmt --check`, `cargo clippy --tests -- -D warnings`, `cargo test`
  (37/37) all pass.

### 4. `--select` is silently ignored

**Status:** todo

**Finding:** `--mouse` warns that it is unsupported on Wayland, `--select`
says nothing and silently captures the whole output or the focused window.

**Cause:** to be confirmed with the fix — `get_window`
(`src/wayland/mod.rs:54`) only warns for `flag.mouse`.

**Fix:** to be filled in.

**Verification:** to be filled in.

### 5. Ctrl-C during the countdown reports an error

**Status:** todo

**Finding:** interrupting during the countdown, before any frame is captured,
exits 1 with `Frame error: No frames found to save` instead of reporting a
cancelled recording.

**Cause:** to be confirmed with the fix — the Ctrl-C handler is installed
before `show_countdown()` (`src/record/mod.rs:121-124`), and an empty frame
list reaches the encoder, which has no way to tell "cancelled" from "broken".
The same applies to the cancel-key path, which clears the frames and breaks.

**Fix:** to be filled in.

**Verification:** to be filled in.

### 6. `MENYOKI_WINDOW_SYSTEM=x11` with no X display panics

**Status:** todo

**Finding:** with no `DISPLAY`, forcing the X11 backend aborts with
`Could not connect to a X display` and dumps core instead of exiting with an
error. The Wayland docs point users at that variable for XWayland windows, so
it is reachable from a Wayland session.

**Cause:** to be confirmed with the fix — not the X11 backend itself:
`x11::display::Display::open` already returns `None` and is handled. The
panic is in `DeviceState::new()` (`device_query`, no `checked_new` on Linux),
called from `InputState::new` via `AppSettings::get_input_state`
(`src/settings.rs:131`) before any window system is touched.

**Fix:** to be filled in.

**Verification:** to be filled in.

### Still not covered by this work

Multi-monitor, fractional scaling and rotated outputs, and non-Hyprland
wlroots compositors stay untested — no hardware here. Fix 3 does change what
a rotated output does: a frame that does not cover the requested area becomes
an error instead of a silently wrong crop.
