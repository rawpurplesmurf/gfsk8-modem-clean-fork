# Changelog

All notable changes to this fork are documented in this file.

## [Unreleased]

### Security

- Patched vendored `vendor/kissfft` (`kiss_fft.c`, `kiss_fftr.c`) for `CVE-2025-34297` (`nfft <= 0` and `SIZE_MAX` integer-overflow bounds validation before `malloc`) and removed `-w` warning suppression in `CMakeLists.txt`.
- Gated standalone PortAudio demo executables behind `option(GFSK8_BUILD_APPS OFF)`.

### Fixed

- Replaced non-standard `constexpr std::round(...)` calls in `src/js8codec.cpp` and `src/JS8.cpp` with ISO C++20 constant expressions so both GCC and Clang compile cleanly.
- Moved C++ test sources from `tests/` to `cpp_tests/` for CodeWizard CI compatibility while keeping `ctest` targets intact, and added `CVE-2025-34297` allocator regression assertions to `test_encode`.
