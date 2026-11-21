# gfsk8-modem-clean-fork

A hardened, dependency-free C++20 static library implementing the **JS8 / GFSK8 weak-signal digital mode modem** (`libgfsk8modem.a` + vendored `libkissfft.a`). Built from first principles for full physical-layer interoperability with JS8Call and linked as a sibling library by [`jf8call-fork`](../jf8call-fork).

- **Zero external dependencies** — no Qt, no FFTW3, no Boost, no Eigen. Single-precision FFTs run on a security-patched vendored copy of KissFFT (`vendor/kissfft/`).
- **Full 5-submode TX & RX** — Normal (15 s), Fast (10 s), Turbo / JS8 40 (6 s), Slow (30 s), and Ultra / JS8 60 (4 s).
- **Complete protocol stack** — Varicode + JSC word compression, 79-symbol 8-FSK with `(174, 87)` LDPC FEC and CRC-12, Costas sync, multi-pass signal subtraction, pilot-driven frequency/timing tracking, noise whitening, LLR soft combining, and LDPC feedback refinement.
- **Security hardened** — vendored KissFFT patched for `CVE-2025-34297`, compiled cleanly under ISO C++20 / C11 with `-Wall -Wextra` on GCC and Clang.

## About This Fork

This repository is a hardened, cleaned-up fork of [`jfrancis42/gfsk8-modem-clean`](https://github.com/jfrancis42/gfsk8-modem-clean) by Jeff Francis (N0GQ) (itself the successor to [`jfrancis42/gfsk8-modem`](https://github.com/jfrancis42/gfsk8-modem)), maintained as the core JS8/GFSK8 modem library alongside [`jf8call-fork`](../jf8call-fork) (our fork of [`jfrancis42/jf8call`](https://github.com/jfrancis42/jf8call)), [`olivia-modem-fork`](../olivia-modem-fork), and [`libpsk-fork`](../libpsk-fork). Key fixes and hardening improvements in this fork include:

- **Security (`CVE-2025-34297`):** Patched vendored `vendor/kissfft` (`kiss_fft.c`, `kiss_fftr.c`) with `nfft <= 0`, odd real-FFT, and `SIZE_MAX` integer-overflow bounds checks prior to `malloc`, with allocator regression assertions in `cpp_tests/test_encode.cpp`.
- **Clean ISO C++20 / C11 Builds:** Removed `-w` warning suppression and non-standard `constexpr std::round()` calls so both GCC and Clang build warning-free under `-Wall -Wextra`.
- **Exact UTC Decode Alignment:** Fixed decode window alignment for non-Normal submodes and derived `kpos` from the caller-supplied capture timestamp (`nutc`) rather than wall-clock execution time.
- **Minimal Dependency Surface & CI Hygiene:** Gated optional PortAudio demo apps behind `GFSK8_BUILD_APPS=OFF`, moved C++ tests to `cpp_tests/`, and added a full documentation suite under [`docs/`](docs/).

---

## Documentation

| Document | Contents |
|----------|----------|
| [API Reference](docs/api-reference.md) | Public `gfsk8` C++ API (`include/gfsk8modem.h`), critical caveats (`Submode` vs. `Decoder` bitmask, `nutc` timestamping, thread safety), and internal APIs (`DecodedText`, `Varicode`, `GFSK8`). |
| [Architecture](docs/architecture.md) | Layered system design, repository file map, active vs. alternate codec translation units, compile-time tables, and FFT plan lifecycle. |
| [DSP & Protocol Specification](docs/dsp-and-protocol.md) | 8-FSK wire format, `(174, 87)` LDPC + CRC-12, Costas arrays, submode DSP parameters, 3-pass RX demodulation pipeline, and Varicode/JSC message framing. |
| [Configuration & Tuning](docs/configuration-and-tuning.md) | All 9 runtime environment variables, diagnostic logging callback, ring-buffer scheduling, and host application integration patterns. |
| [Building, Testing & Security](docs/building-testing-and-security.md) | CMake build options, `cpp_tests/` verification targets, `test_decode_wav` CLI usage, `CVE-2025-34297` hardening details, and PR workflow. |
| [Changelog](CHANGELOG.md) | History of security fixes, build updates, and modem bug fixes in this fork. |

---

## Quick Start

### Prerequisites

- CMake ≥ 3.20
- A C++20 / C11 compiler (GCC 11+ or Clang 14+)

### Build and Test

```bash
git clone <repo-url> gfsk8-modem-clean-fork
cd gfsk8-modem-clean-fork
cmake -B build -S . -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
ctest --test-dir build --output-on-failure
```

Build outputs:

- `build/libgfsk8modem.a` — static JS8/GFSK8 modem library
- `build/libkissfft.a` — static vendored KissFFT library
- `build/test_encode`, `build/test_loopback`, `build/test_loopback_all` — CTest test binaries
- `build/test_decode_wav` — standalone 12 kHz mono 16-bit PCM WAV decoder CLI

### CMake Integration

```cmake
add_subdirectory(path/to/gfsk8-modem-clean-fork)
target_link_libraries(your_target PRIVATE gfsk8modem)
```

CMake automatically propagates the public include directory (`include/`) and links the private vendored `kissfft` archive.

---

## Usage Examples

Include `include/gfsk8modem.h`. All public symbols live in `namespace gfsk8`.

### Pack, Encode, and Modulate (TX)

```cpp
#include "gfsk8modem.h"
#include <vector>

// 1. Pack a human-readable JS8 message into one or more 12-char physical frames.
auto frames = gfsk8::pack("W5XYZ", "DM79AA", "CQ CQ CQ", gfsk8::Submode::Normal);

for (auto const &frame : frames) {
    // 2. Modulate to 12 kHz float32 PCM in [-1.0, 1.0] (includes start-delay silence).
    std::vector<float> pcm = gfsk8::modulate(
        gfsk8::Submode::Normal,
        frame.frameType,
        frame.payload,
        1500.0 /* carrier frequency in Hz */
    );
    // Transmit pcm in consecutive UTC-aligned periods.
}
```

### Decode a 60-Second Audio Snapshot (RX)

```cpp
#include "gfsk8modem.h"
#include <chrono>
#include <cstdio>
#include <span>

// Create once and reuse across decode periods (allocates FFT plans on construction).
// Pass a bitmask of submodes: 1=Normal, 2=Fast, 4=Turbo, 8=Slow, 16=Ultra, 0x1F=All.
gfsk8::Decoder decoder(gfsk8::AllSubmodes, /*nfa=*/200, /*nfb=*/4000);

// At each UTC period boundary, snapshot 60 s (720,000 samples) of 12 kHz int16 audio
// with oldest sample at index 0 and newest sample at index 719,999.
// Pass capture-time UTC seconds since midnight (H*3600 + M*60 + S).
auto epochSec = std::chrono::duration_cast<std::chrono::seconds>(
    std::chrono::system_clock::now().time_since_epoch()).count();
int nutc = static_cast<int>(epochSec % 86400);

decoder.decode(audioSnapshot, nutc, [](gfsk8::Decoded const &d) {
    std::printf("submode=%d snr=%+d dB freq=%.1f Hz dt=%+.2f s q=%.2f raw=[%s]\n",
                d.submode, d.snrDb, d.frequencyHz, d.dtSeconds,
                d.quality, d.message.c_str());
});
```

> **Important notes for integrators:**
>
> 1. **Bitmask vs. Enum:** `gfsk8::Submode` enum values (`0, 1, 2, 4, 8`) are **not** the same as the `Decoder` constructor bitmask bits (`1, 2, 4, 8, 16`). See [API Reference — Submode Enum vs. Decoder Bitmask](docs/api-reference.md#submode-enum-vs-decoder-bitmask).
> 2. **`nutc` format:** Pass UTC seconds since midnight (`0..86399`), captured at the end of the audio buffer, so UTC-aligned decode windows stay exact.
> 3. **Thread safety:** `Decoder::decode()` writes through a shared internal staging buffer (`dec_data`); serialize `decode()` calls across threads.

---

## JS8 Submode Summary

All submodes operate at **12,000 Hz** audio sample rate and transmit **79 8-FSK symbols** per frame (21 Costas sync symbols + 58 data/parity symbols).

| Submode | Band ID | `gfsk8::Submode` | `Decoder` Bit | Period | Samples/Sym | Tone Spacing | Bandwidth | Start Delay | SNR Threshold | Costas Array |
|---------|---------|------------------|---------------|--------|-------------|--------------|-----------|-------------|---------------|--------------|
| Normal | JS8A | `0` | `1 << 0` (`1`) | 15 s | 1920 | 6.25 Hz | 50 Hz | 500 ms | −24 dB | Original |
| Fast | JS8B | `1` | `1 << 1` (`2`) | 10 s | 1200 | 10.00 Hz | 80 Hz | 200 ms | −22 dB | Modified |
| Turbo (JS8 40) | JS8C | `2` | `1 << 2` (`4`) | 6 s | 600 | 20.00 Hz | 160 Hz | 100 ms | −20 dB | Modified |
| Slow | JS8E | `4` | `1 << 3` (`8`) | 30 s | 3840 | 3.125 Hz | 25 Hz | 500 ms | −28 dB | Modified |
| Ultra (JS8 60) | JS8I | `8` | `1 << 4` (`16`) | 4 s | 384 | 31.25 Hz | 250 Hz | 100 ms | −18 dB | Modified |

---

## License & Attribution

Licensed under **GPL-3.0**. Any project linking `libgfsk8modem.a` must comply with GPL-3.0 or a compatible license.

- **WSJT-X / FT8 DSP & LDPC foundations:** Joe Taylor (K1JT) and the WSJT-X development team.
- **JS8 protocol, Varicode, and JSC compression:** Jordan Sherer (KN4CRD).
- **C++ port of the Fortran JS8 modem:** Allan Bazinet (W6BAZ).
- **Original Standalone Library Extraction:** Forked from [`jfrancis42/gfsk8-modem-clean`](https://github.com/jfrancis42/gfsk8-modem-clean) (successor to [`jfrancis42/gfsk8-modem`](https://github.com/jfrancis42/gfsk8-modem)) by **Jeff Francis (N0GQ)**.
- **Vendored FFT Engine:** KissFFT by Mark Borgerding (BSD-3-Clause), patched in this fork for `CVE-2025-34297`.
