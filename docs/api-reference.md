# API Reference

This document covers the public C++20 interface exposed in `include/gfsk8modem.h` (`namespace gfsk8`), critical integration contracts, and the internal headers available to tightly coupled callers (such as [`jf8call-fork`](../../jf8call-fork)).

---

## Public Header: `include/gfsk8modem.h`

All public symbols reside in `namespace gfsk8`.

### Submodes & Parameters

#### `enum class Submode : int`

Identifies one of the five JS8 submodes. The numeric values match `Varicode::SubmodeType` from JS8Call-improved:

```cpp
enum class Submode : int {
    Normal = 0,  // 15 s period,  6.25 Hz tone spacing (JS8A)
    Fast   = 1,  // 10 s period, 10.00 Hz tone spacing (JS8B)
    Turbo  = 2,  //  6 s period, 20.00 Hz tone spacing (JS8C / "JS8 40")
    Slow   = 4,  // 30 s period,  3.125 Hz tone spacing (JS8E)
    Ultra  = 8,  //  4 s period, 31.25 Hz tone spacing (JS8I / "JS8 60")
};

static constexpr int AllSubmodes = 0x1F;
```

#### Submode Enum vs. Decoder Bitmask

> **Warning:** Do **not** pass `static_cast<int>(Submode::*)` directly as the `submodes` bitmask argument to `gfsk8::Decoder::Decoder()`.

`Submode` enumerator values are protocol IDs (`0, 1, 2, 4, 8`), whereas `Decoder` checks bit positions `0..4` (`1 << shift`):

| Submode | `gfsk8::Submode` value | `Decoder` constructor bitmask |
|---------|------------------------|-------------------------------|
| Normal (ModeA) | `0` | `1 << 0` = `1` (`0x01`) |
| Fast (ModeB) | `1` | `1 << 1` = `2` (`0x02`) |
| Turbo (ModeC) | `2` | `1 << 2` = `4` (`0x04`) |
| Slow (ModeE) | `4` | `1 << 3` = `8` (`0x08`) |
| Ultra (ModeI) | `8` | `1 << 4` = `16` (`0x10`) |
| All 5 submodes | — | `gfsk8::AllSubmodes` (`0x1F`) |

Passing `static_cast<int>(Submode::Normal)` (`0`) enables **zero** submodes; passing `static_cast<int>(Submode::Fast)` (`1`) actually enables **Normal**. Combine explicit bit shifts instead:

```cpp
constexpr int kNormalAndFast = (1 << 0) | (1 << 1); // Normal + Fast
gfsk8::Decoder dec(kNormalAndFast);
```

Note that `Decoded::submode` returned in the RX callback uses the `Submode` enum values (`0, 1, 2, 4, 8`), **not** the constructor bitmask values.

#### `struct SubmodeParms` and `submodeParms()`

```cpp
struct SubmodeParms {
    int    samplesPerSymbol;  // audio samples per symbol at 12,000 Hz
    double toneSpacingHz;     // Hz between adjacent 8-FSK tones (12000.0 / samplesPerSymbol)
    int    periodSeconds;     // total TX/RX slot duration in seconds
    int    startDelayMs;      // silence prefix before symbol 0 (ms)
    int    numSymbols;        // always 79
    int    sampleRate;        // always 12000 Hz
    int    rxSnrThresholdDb;  // nominal decode threshold in dB
};

SubmodeParms submodeParms(Submode s);
```

Throws `GFSK8::Submode::error` (deriving from `std::runtime_error`) if an invalid integer is cast to `Submode`.

---

### Transmit (TX) Functions

#### `pack()`

```cpp
struct TxFrame {
    std::string payload;   // exactly 12-char JS8 physical-layer alphabet payload
    int         frameType; // lower 3 bits: Varicode::TransmissionType flags
};

std::vector<TxFrame> pack(std::string const &mycall,
                          std::string const &mygrid,
                          std::string const &text,
                          Submode submode = Submode::Normal);
```

Packs a human-readable JS8 message into one or more 12-character physical-layer frames using `Varicode::buildMessageFrames()`.

- Supports heartbeat/CQ frames, directed frames (including automatic CRC-16/CRC-32 checksums for buffered commands except `@APRSIS`), compound callsign frames, and multi-frame Varicode/JSC data payloads.
- Sets `Varicode::JS8CallFirst` (`1`) on the first frame, `Varicode::JS8CallLast` (`2`) on the last frame, and `Varicode::JS8CallData` (`4`) on non-Normal fast-data frames.
- Returns an empty vector if the input cannot be packed.
- Multi-frame messages should be transmitted one frame per successive UTC-aligned period.

#### `encode()`

```cpp
bool encode(Submode submode, int frameType,
            std::string_view message,
            std::vector<int> &tonesOut);
```

Physical-layer encoder (`GFSK8::encode` / Fortran `genjs8` port).

- `submode`: Selects the Costas array (`ORIGINAL` for Normal, `MODIFIED` for Fast/Turbo/Slow/Ultra).
- `frameType`: Lower 3 bits (`frameType & 0b111`) are embedded as bits 72–74 of the 87-bit codeword.
- `message`: Must be **exact length 12** using only characters from the 64-character JS8 physical alphabet:
  `0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz-+` (no spaces).
- `tonesOut`: Resized to 79 integers, each in `[0, 7]`.
- Returns `false` if `message.size() != 12` or any character is outside the 64-character alphabet.

#### `modulate()`

```cpp
std::vector<float> modulate(Submode submode, int frameType,
                            std::string_view message,
                            double audioFrequencyHz = 1500.0);
```

Calls `encode()` and synthesizes continuous-phase 8-FSK PCM audio at **12,000 Hz** (`float32` in `[-1.0, +1.0]`).

- Output length is `round(startDelayMs * 12) + 79 * samplesPerSymbol` samples:
  - Normal: `6,000 + 79 × 1,920 = 157,680` samples (13.14 s)
  - Fast: `2,400 + 79 × 1,200 = 97,200` samples (8.10 s)
  - Turbo: `1,200 + 79 × 600 = 48,600` samples (4.05 s)
  - Slow: `6,000 + 79 × 3,840 = 309,360` samples (25.78 s)
  - Ultra: `1,200 + 79 × 384 = 31,536` samples (2.628 s)
- Begins with `startDelayMs` of exact zeros, followed by continuous-phase sine waves at `audioFrequencyHz + tone * toneSpacingHz`.
- Returns an empty vector on encoding error.

---

### Receive (RX) Class: `Decoder`

#### `struct Decoded` and `DecodeCallback`

```cpp
struct Decoded {
    std::string message;    // 12-character raw physical-layer payload
    int    snrDb;           // estimated SNR in dB (clamped >= -60 dB)
    float  frequencyHz;     // refined carrier frequency (Hz)
    float  dtSeconds;       // timing offset relative to submode start delay (s)
    int    submode;         // Submode enum value (0, 1, 2, 4, or 8)
    float  quality;         // 1.0f - (nharderrors / 60.0f) in (0.0, 1.0]
    int    frameType;       // 3-bit TransmissionType field (bits 72..74)
    int    utc;             // echoes the nutc passed into decode()
};

using DecodeCallback = std::function<void(Decoded const &)>;
```

#### `Decoder` Lifecycle

```cpp
class Decoder {
public:
    explicit Decoder(int submodes = AllSubmodes, int nfa = 200, int nfb = 4000);
    ~Decoder();

    Decoder(Decoder const &) = delete;
    Decoder &operator=(Decoder const &) = delete;
    Decoder(Decoder &&) = delete;
    Decoder &operator=(Decoder &&) = delete;

    void decode(std::span<std::int16_t const> samples,
                int nutc,
                DecodeCallback const &cb);

    void reset();
};
```

- **Constructor:** Allocates all five `DecodeMode<Mode>` state objects and their KissFFT plans (6 plans per mode = 30 FFT plans total). Construction is expensive; instantiate once and reuse across decode periods.
  - `submodes`: Bitmask of enabled submodes (`1=Normal`, `2=Fast`, `4=Turbo`, `8=Slow`, `16=Ultra`, `0x1F=All`).
  - `nfa`, `nfb`: Low and high decode passband bounds in Hz (internally clamped to `[100, 4910]` Hz).
- **`decode(samples, nutc, cb)`:**
  - Accepts 12 kHz mono `int16_t` PCM samples representing a linear snapshot of a 60-second ring buffer (`720,000` samples = `GFSK8_RX_SAMPLE_SIZE`), ordered oldest (`samples[0]`) to newest (`samples[samples.size() - 1]`).
  - If `samples.size() < 720,000`, the remainder is zero-padded at the end; if longer, it is truncated to `720,000`. Callers with shorter recordings should right-align audio at the end of a 720,000-sample buffer (see `cpp_tests/test_decode_wav.cpp`).
  - Invokes `cb` synchronously on the calling thread for each unique decoded frame (or improved-SNR duplicate within the same pass).
- **`reset()`:** Destroys and recreates the internal `GFSK8::Decoder`, clearing all per-mode `SoftCombiner` caches.

#### `nutc` Timestamp & UTC Window Alignment

Although the doc comment in `include/gfsk8modem.h` historically mentions `code_time()`, **`src/api.cpp` expects `nutc` to be UTC seconds since midnight** (`H * 3600 + M * 60 + S`, range `0..86399`), captured at the end of the audio snapshot:

```cpp
int const sec_in_minute = nutc % 60;
int const elapsed       = sec_in_minute % period_sec;
int const kpos          = std::max(0, 720000 - (elapsed + period_sec) * 12000);
```

This aligns every enabled submode's window `[kpos, kpos + period_sec * 12000)` to the **most recently completed UTC-aligned slot** using the capture timestamp rather than wall-clock execution time (preventing missed decodes when worker threads lag on loaded systems).

> **Note:** If you pass `code_time(H, M, S)` (`HHMMSS`, e.g., `123456`) instead of seconds since midnight, `nutc % 60` will not equal `S` (since `100 % 60 == 40`), misaligning non-zero-minute windows. Always pass `epoch_seconds % 86400` (or `H * 3600 + M * 60 + S`). For offline WAV decoding where the audio is right-aligned to the end of the 720,000-sample buffer and ends on a period boundary, pass `nutc = 0` (`elapsed = 0`, placing the window at the final `period_sec` seconds of the buffer).

---

### Logging

```cpp
void setLogCallback(std::function<void(const char *)> cb);
```

Registers a process-global diagnostic callback (`gfsk8::g_logger`) used by `GFSK8_LOG(...)`. When unset, messages are silently discarded. Set once at startup before invoking other APIs. Note that Qt-origin `qCDebug` / `qCWarning` / `qCInfo` statements inside the decoder expand to compile-time no-ops (`DiagSink`).

---

## Thread Safety & Global State

The library maintains four process-wide globals defined in `src/api.cpp`:

| Global | Type | Purpose |
|--------|------|---------|
| `dec_data` | `RxAudioBuffer` | Staging buffer (`d2[720000]` + `DecodeParams`) written by `Decoder::decode()` before `snapshot()`. |
| `specData` | `RxSpectrum` | Spectrum output arrays (`savg`, `slin`). |
| `fftw_mutex` | `std::mutex` | Guards KissFFT plan creation and destruction. |
| `gfsk8::g_logger` | `gfsk8::LogDispatcher` | Holds the optional user log callback. |

**Concurrency rules:**

1. **Single `Decoder` instance:** Not thread-safe; never call `decode()` or `reset()` concurrently on the same instance.
2. **Multiple `Decoder` instances:** Because `gfsk8::Decoder::decode()` copies into the unsynchronized process-global `dec_data` prior to calling `m_impl->core->snapshot()`, callers with multiple `Decoder` instances across threads **must guard `decode()` with an external mutex** (or serialize decode jobs through a single worker queue, as `jf8call-fork` does).
3. **`encode()`, `modulate()`, `submodeParms()`:** Stateless and thread-safe.
4. **`pack()`:** Uses `JSC::lookup()`, which populates a process-global `static std::unordered_map` (`s_wordCache` in `src/JSC.cpp`) without locking; synchronize concurrent `pack()` calls across threads.

---

## Internal APIs (Private `src/` Headers)

The public CMake include path exposes only `include/gfsk8modem.h`. Projects that need RX text unpacking or direct access to lower-level utilities can include internal headers from `src/` (as `cpp_tests/test_decode_wav.cpp` does):

| Header | Primary Types / Functions | Use Case |
|--------|---------------------------|----------|
| `src/DecodedText.h` | `class DecodedText` | Unpacks a 12-char physical payload (`Decoded::message`, `frameType`, `submode`) into human-readable JS8 text (`message()`, `isHeartbeat()`, `isDirectedMessage()`, `isCompound()`, `isLowConfidence()`). |
| `src/Varicode.h` | `class Varicode` | Low-level JS8 Varicode/Huffman coding, callsign/grid/command bit packing, CRC-16/KERMIT & CRC-32/BZIP2 checksums, `buildMessageFrames()`. |
| `src/JSC.h` | `class JSC` | 262,144-entry `(s, c)`-dense word compressor/decompressor. |
| `src/JS8Submode.h` | `namespace GFSK8::Submode` | Detailed submode queries (`bandwidth`, `costas`, `samplesNeeded`, `rxThreshold`, `dataDuration`, `txDuration`, `computeCycleForDecode`). |
| `src/JS8.h` | `GFSK8::Decoder`, `GFSK8::Event::Variant` | Low-level event-emitting decoder exposing `DecodeStarted`, `SyncStart`, `SyncState`, `Decoded`, and `DecodeFinished`. |
| `src/commons.h` | `GFSK8_RX_SAMPLE_RATE`, `GFSK8_RX_SAMPLE_SIZE`, `RxAudioBuffer` | Core sample-rate/buffer macros (`12000`, `720000`) and shared parameter layout. |

### Unpacking a Decoded Frame with `DecodedText`

```cpp
#include "gfsk8modem.h"
#include "DecodedText.h" // from src/

decoder.decode(audioSnapshot, nutc, [](gfsk8::Decoded const &d) {
    DecodedText dt(d.message, d.frameType, d.submode);
    std::printf("raw=[%s]  text=[%s]  hb=%d  directed=%d\n",
                d.message.c_str(), dt.message().c_str(),
                dt.isHeartbeat(), dt.isDirectedMessage());
});
```

Frames whose `quality` is below `0.17f` (`kQualityThreshold`) are flagged `isLowConfidence() == true` when constructed from `GFSK8::Event::Decoded`.
