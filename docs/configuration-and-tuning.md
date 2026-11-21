# Configuration, Diagnostics & Integration Guide

This guide documents all runtime environment variables, diagnostic logging hooks, and best practices for embedding `gfsk8-modem-clean-fork` in a real-time receiver/transmitter such as [`jf8call-fork`](../../jf8call-fork) or a headless daemon.

---

## Runtime Environment Variables

The decoder supports nine environment variables for tuning or A/B-testing individual DSP stages without recompiling.

| Environment Variable | Default | Read Time | Effect |
|----------------------|---------|-----------|--------|
| `GFSK8_DISABLE_FREQ_TRACKING` | Enabled (unset) | `Decoder` construction | Any value disables pilot-driven `FrequencyTracker` during symbol extraction. |
| `GFSK8_DISABLE_TIMING_TRACKING` | Enabled (unset) | `Decoder` construction | Any value disables early/late Goertzel `TimingTracker` during symbol extraction. |
| `GFSK8_DISABLE_WHITENING` | Enabled (unset) | Every candidate decode | Any value disables per-tone / per-symbol median noise whitening in `WhiteningProcessor`. (Erasure thresholding still runs as a fallback pass unless also disabled.) |
| `GFSK8_LLR_ERASURE_THRESH` | `0.25` | `Decoder` construction | Float threshold below which `\|LLR\|` is zeroed (erased). Non-positive or non-finite values disable erasure thresholding (`0.0`). |
| `GFSK8_DISABLE_ERASURE_THRESHOLDING` | Enabled (unset) | `Decoder` construction | Any value forces the erasure threshold to `0.0`, disabling LLR zeroing. |
| `GFSK8_LDPC_FEEDBACK` | `1` (enabled) | `Decoder` construction | Set to `0` to disable iterative LLR feedback refinement between BP LDPC attempts (`atoi(val) != 0`). |
| `GFSK8_LDPC_MAX_PASSES` | `8` | `Decoder` construction | Maximum total `bpdecode174` executions per candidate across all erasure + feedback passes. Clamped to `[1, 8]`. |
| `GFSK8_SOFT_COMBINING` | `1` (enabled) | `Decoder` construction | Set to `0` to disable cross-period LLR soft combining (`atoi(val) != 0`). |
| `GFSK8_SOFT_COMBINING_TEST` | Disabled (unset) | First `SoftCombiner` construction | Any value triggers a one-time (`std::call_once`) synthetic soft-combining self-test. |

### Tuning Profiles

- **Lowest CPU latency (e.g., constrained ARM boards):**
  ```bash
  GFSK8_LDPC_MAX_PASSES=4 GFSK8_LDPC_FEEDBACK=0 ./your_app
  ```
- **Pure baseline comparison (disabling modern DSP enhancements):**
  ```bash
  GFSK8_DISABLE_FREQ_TRACKING=1 \
  GFSK8_DISABLE_TIMING_TRACKING=1 \
  GFSK8_DISABLE_WHITENING=1 \
  GFSK8_DISABLE_ERASURE_THRESHOLDING=1 \
  GFSK8_LDPC_FEEDBACK=0 \
  GFSK8_SOFT_COMBINING=0 ./your_app
  ```

Because most environment variables are latched inside `DecodeMode`'s constructor, set them **before** constructing `gfsk8::Decoder` (or call `Decoder::reset()` after modifying environment variables).

---

## Diagnostic Logging

Call `gfsk8::setLogCallback()` once at startup to capture messages emitted through `GFSK8_LOG(...)`:

```cpp
#include "gfsk8modem.h"
#include <cstdio>

gfsk8::setLogCallback([](const char *msg) {
    std::fprintf(stderr, "[gfsk8] %s\n", msg);
});
```

- If no callback is registered, `GFSK8_LOG` formats nothing and incurs a single null-check branch.
- Qt-derived `qCDebug(decoder_js8)` / `qCWarning` / `qCInfo` calls inside `JS8.cpp`, `soft_combiner.h`, and `Varicode.cpp` compile away to no-ops (`DiagSink` in `src/log.h`).

---

## Host Application Integration Patterns

### 1. Audio Pipeline & Resampling

- **RX:** Most soundcards (PipeWire, PulseAudio, ALSA, PortAudio) run natively at **48,000 Hz**. Decimate 4:1 (with an appropriate anti-aliasing low-pass filter, e.g., cutoff ~5 kHz) to **12,000 Hz** mono `int16_t` before pushing samples into your 60-second ring buffer (`720,000` samples).
- **TX:** `gfsk8::modulate()` returns `float32` mono PCM at **12,000 Hz** in `[-1.0, +1.0]`. Interpolate 1:4 to **48,000 Hz** (or your device rate) and apply desired TX audio gain / ALC headroom before playback.

### 2. Ring-Buffer Snapshot & UTC Scheduling

Maintain a rolling 60-second circular buffer of 12 kHz `int16_t` samples:

1. Continuously append incoming 12 kHz audio into the circular buffer.
2. When a UTC period boundary expires for one or more submodes:
   - Normal (15 s): `sec % 15 == 0` (`:00, :15, :30, :45`)
   - Fast (10 s): `sec % 10 == 0` (`:00, :10, :20, :30, :40, :50`)
   - Turbo (6 s): `sec % 6 == 0` (`:00, :06, :12, ...`)
   - Slow (30 s): `sec % 30 == 0` (`:00, :30`)
   - Ultra (4 s): `sec % 4 == 0` (`:00, :04, :08, ...`)
3. Linearize the ring buffer into a contiguous `720,000`-sample snapshot (oldest at index `0`, newest at index `719,999`).
4. **Capture `nutc` at snapshot time** — *not* inside the background decode thread:
   ```cpp
   auto nowSec = std::chrono::duration_cast<std::chrono::seconds>(
       std::chrono::system_clock::now().time_since_epoch()).count();
   int nutc = static_cast<int>(nowSec % 86400);
   ```
5. Dispatch `(snapshot, nutc, submodeMask)` to a single serialized decode worker thread.

### 3. Selective Submode Decoding Pattern (as used in `jf8call-fork`)

Because `gfsk8::Decoder` fixes its `submodes` bitmask at construction, decoding only the submodes whose UTC period just completed avoids redundant work (e.g., not re-decoding Slow every 4 seconds):

- Keep one `gfsk8::Decoder(gfsk8::AllSubmodes)` for full-suite passes, **or** lazily maintain up to five single-bit decoders (`1 << 0` through `1 << 4`), invoking only the decoders whose bit is set in the boundary mask — always on the **same worker thread** (or protected by one mutex because `dec_data` is process-global).

### 4. Mapping UI Indices to `gfsk8::Submode`

Many frontends index submodes sequentially (`0..4`). Keep an explicit lookup table rather than casting indices directly:

```cpp
static constexpr gfsk8::Submode kUiToSubmode[5] = {
    gfsk8::Submode::Normal, // UI 0 -> enum 0, Decoder bit 1 << 0 (1)
    gfsk8::Submode::Fast,   // UI 1 -> enum 1, Decoder bit 1 << 1 (2)
    gfsk8::Submode::Turbo,  // UI 2 -> enum 2, Decoder bit 1 << 2 (4)
    gfsk8::Submode::Slow,   // UI 3 -> enum 4, Decoder bit 1 << 3 (8)
    gfsk8::Submode::Ultra,  // UI 4 -> enum 8, Decoder bit 1 << 4 (16)
};
```
