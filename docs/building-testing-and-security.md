# Building, Testing & Security

This document covers CMake build configuration, automated and manual testing, security hardening in `gfsk8-modem-clean-fork`, and the repository's contribution workflow.

---

## Building

### Requirements

- **CMake:** ≥ 3.20
- **C++ Compiler:** ISO C++20 (`CMAKE_CXX_STANDARD 20`, `CMAKE_CXX_STANDARD_REQUIRED ON`)
- **C Compiler:** ISO C11 (`CMAKE_C_STANDARD 11`)
- **External Libraries:** None required for the modem library and test suite.

### Standard Release Build

```bash
cmake -B build -S . -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

Both `kissfft` and `gfsk8modem` are compiled with `-Wall -Wextra` (warning suppression is intentionally disallowed). Non-standard compiler extensions such as `constexpr std::round()` have been removed so both GCC and Clang compile cleanly.

### CMake Options

| Option | Default | Description |
|--------|---------|-------------|
| `GFSK8_BUILD_APPS` | `OFF` | Enables optional PortAudio demo executables (`js8rx`, `js8tx`) if `apps/` sources are present and `portaudio-2.0` / `portaudio` is found via `pkg-config`. Disabled by default so library builds remain dependency-free. |

### Produced Targets

| Target | Artifact | Description |
|--------|----------|-------------|
| `kissfft` | `build/libkissfft.a` | Static library compiled from `vendor/kissfft/kiss_fft.c` and `kiss_fftr.c`. |
| `gfsk8modem` | `build/libgfsk8modem.a` | Static JS8/GFSK8 modem library linking privately against `kissfft`. |
| `test_encode` | `build/test_encode` | Registered with CTest as `encode`. |
| `test_loopback` | `build/test_loopback` | Registered with CTest as `loopback`. |
| `test_loopback_all` | `build/test_loopback_all` | Registered with CTest as `loopback_all`. |
| `test_decode_wav` | `build/test_decode_wav` | Standalone CLI WAV decoder (not registered in CTest). |

---

## Testing

### Running the Automated Test Suite

```bash
ctest --test-dir build --output-on-failure
```

| CTest Name | Source File | Coverage |
|------------|-------------|----------|
| `encode` | `cpp_tests/test_encode.cpp` | Encodes `"HELLOWORLD--"` across all 5 submodes (checking 79 tones in `[0, 7]`, sample rate, and `SubmodeParms`), verifies `modulate(Normal)` exact PCM length (`157,680` samples), and runs `CVE-2025-34297` regression checks against `kiss_fft_alloc` and `kiss_fftr_alloc`. |
| `loopback` | `cpp_tests/test_loopback.cpp` | Modulates `"TESTTEST1234"` in Normal mode at 1500 Hz, scales to `int16`, places it at the UTC-aligned `kposA` derived from `nutc = epoch_sec % 86400`, decodes with `Decoder(1 << 0)`, and asserts exact message recovery. |
| `loopback_all` | `cpp_tests/test_loopback_all.cpp` | Runs full `pack("K1TEST", "DM79", "HELLO", sm)` → `modulate()` → UTC-aligned snapshot → `Decoder(bit).decode()` across all 5 submodes (Normal, Fast, Turbo, Slow, Ultra). |

> **Why `cpp_tests/` instead of `tests/`?** Keeping C++ test sources in `cpp_tests/` prevents Python `pytest` CI auto-discovery from scanning C++ files and exiting with code 5 ("no tests collected").

### Decoding a WAV File Manually (`test_decode_wav`)

`test_decode_wav` decodes a **12,000 Hz, 16-bit, mono PCM WAV** file and prints both the raw 12-character physical frame and the human-readable `DecodedText`-unpacked message:

```bash
./build/test_decode_wav recording.wav [submode_mask]
```

- `submode_mask` defaults to `31` (`gfsk8::AllSubmodes`).
- Note that `test_decode_wav` passes `submode_mask` directly to `gfsk8::Decoder(submode_mask)`, so use the `Decoder` bitmask values (`1`=Normal, `2`=Fast, `4`=Turbo, `8`=Slow, `16`=Ultra, `31`=All).
- The WAV samples are right-aligned at the end of a `720,000`-sample (60 s) zero-padded buffer and decoded with `nutc = 0`.

---

## Security Hardening

### 1. Vendored KissFFT `CVE-2025-34297` Remediation

`vendor/kissfft/kiss_fft.c` and `vendor/kissfft/kiss_fftr.c` have been hardened against integer-overflow heap corruption prior to `malloc`:

- **`kiss_fft_alloc()` (`vendor/kissfft/kiss_fft.c`):**
  - Rejects `nfft <= 0` immediately, returning `NULL`.
  - Validates `(size_t)(nfft - 1) <= SIZE_MAX / sizeof(kiss_fft_cpx)` and checks addition headroom against `sizeof(struct kiss_fft_state) + 64` before computing `memneeded`.
- **`kiss_fft_stride()` (`vendor/kissfft/kiss_fft.c`):**
  - Validates `st != NULL`, `st->nfft > 0`, and `(size_t)st->nfft <= SIZE_MAX / sizeof(kiss_fft_cpx)` before temporary in-place buffer allocation.
- **`kiss_fftr_alloc()` (`vendor/kissfft/kiss_fftr.c`):**
  - Rejects `nfft <= 0` or odd lengths (`(nfft & 1) != 0`), returning `NULL`.
  - Checks `subsize`, `(size_t)nfft <= SIZE_MAX / 3`, `twiddle_count <= SIZE_MAX / sizeof(kiss_fft_cpx)`, and total byte-sum overflow before calling `KISS_FFT_MALLOC`.
- **Regression tests:** `cpp_tests/test_encode.cpp` asserts `NULL` returns for zero, negative, odd real-FFT, and 32-bit `INT_MAX` sizes.

### 2. Minimal Attack Surface

- No networking, telemetry, update checkers, or dynamic plugin loading.
- Zero third-party shared library dependencies (only standard C/C++ runtime and libm).
- Standalone PortAudio demo apps are disabled by default (`GFSK8_BUILD_APPS=OFF`).

### 3. Protocol CRC-12 Integrity

`src/crc12.h` and `vendor/crc12.h` implement the exact MSB-first non-reflected polynomial `0xC06` matching `boost::augmented_crc<12, 0xc06>`, fixing an earlier bug where a `0x80F` CRC cancelled out in loopback tests while rejecting real JS8Call on-air frames.

---

## Upstream Provenance & Fork Relationships

| Repository | Role |
|------------|------|
| **`gfsk8-modem-clean-fork`** (this repo) | Hardened, cleaned-up fork with `CVE-2025-34297` KissFFT patches, ISO C++20/C11 `-Wall -Wextra` builds, UTC window alignment fixes, and full documentation (`libgfsk8modem.a` + `libkissfft.a`). |
| [`jfrancis42/gfsk8-modem-clean`](https://github.com/jfrancis42/gfsk8-modem-clean) | Original upstream standalone JS8 modem by Jeff Francis (N0GQ) (successor to [`jfrancis42/gfsk8-modem`](https://github.com/jfrancis42/gfsk8-modem)). |
| [`jf8call-fork`](../../jf8call-fork) | Primary downstream consumer (`src/gfsk8_modem.h`, `src/gfsk8_modem.cpp`), forked from [`jfrancis42/jf8call`](https://github.com/jfrancis42/jf8call). |
| [`olivia-modem-fork`](../../olivia-modem-fork) / [`libpsk-fork`](../../libpsk-fork) | Sibling zero-dependency modem forks used alongside `gfsk8-modem-clean-fork` in `jf8call-fork`. |

---

## Contributing & PR Workflow

Per [`AGENTS.md`](../AGENTS.md):

1. **Never commit or push directly to `main`.** Always branch off `main` using `feat/*`, `fix/*`, or `docs/*`.
2. **Verify locally before pushing:**
   ```bash
   cmake -B build -S . -DCMAKE_BUILD_TYPE=Release
   cmake --build build -j
   ctest --test-dir build --output-on-failure
   ```
3. **Preserve guardrails:**
   - Keep C++ tests inside `cpp_tests/`.
   - Preserve `CVE-2025-34297` overflow bounds checks in `vendor/kissfft/`.
   - Maintain `-Wall -Wextra` warning-free C++20/C11 builds.
   - Maintain GPL-3.0 license compatibility.
4. **Pull Request & Merge:** Open a PR targeting `main`, wait for automated code review (`APPROVED`) and CI pipeline (`success`), and merge via `rebase-merge`.
