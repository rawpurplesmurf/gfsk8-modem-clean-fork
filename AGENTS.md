# gfsk8-modem-clean-fork — AI Coding Agent Instructions

Hardened C++20 JS8/GFSK8 static modem library (`libgfsk8modem.a` + `libkissfft.a`) used by `dan/jf8call-fork` and Grid Buster.

## CodeWizard PR Workflow

1. Never commit or push directly to `main`; always use an isolated `feat/*`, `fix/*`, or `docs/*` branch targeting `dan/gfsk8-modem-clean-fork` `main` (upstream mirror: `codewizard/gfsk8-modem-clean`).
2. Verify CMake build (`-Wall -Wextra`) and `ctest --test-dir build --output-on-failure` locally before pushing.
3. Sync worktree via `codewizard_pull_repository`, open a PR, wait for `codewizard/code-review` (`APPROVED`) + 7-scanner `codewizard/pipeline` (`success`), and merge with `rebase-merge`.
4. Keep C++ tests in `cpp_tests/` (so Python `pytest` auto-discovery does not exit with code 5 on non-Python files) and preserve `CVE-2025-34297` integer-overflow bounds checks in `vendor/kissfft/`.
