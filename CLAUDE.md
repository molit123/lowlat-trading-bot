# CLAUDE.md: lowlat-gpu

Personal project: low-latency trading systems in C++ plus GPU inference optimization (CUDA).
Goal: hands-on mastery of memory/cache optimization, lock-free design, GPU launch/transfer overheads, and honest latency measurement. Not a production trading system.

## The central question
"At what batch size and latency budget does a GPU beat a CPU for small trading models, and how much of that is launch/transfer overhead?" Every component supports answering that with measured percentiles.

## Hardware / environment
- GPU: NVIDIA RTX 5070 Ti (Blackwell, sm_120). **CUDA toolkit >= 12.8 required.** 16 GB VRAM.
- Dev starts on WSL2 (Ubuntu 24.04), moves to native Ubuntu dual-boot for final latency numbers. See `docs/SETUP.md`.
- Secondary: aarch64 Linux laptop (no CUDA). CPU code must build there: cache-line size is 128 B on that machine, so never hardcode 64.
- Always record environment (kernel, driver, CUDA version, WSL vs native, governor, isolated cores) next to results.

## Ground rules
1. **Benchmark before optimizing.** No optimization lands without a before/after number from the harness.
2. **Correctness first.** Every fast path (SIMD, CUDA kernel, quantized) is diffed against a plain CPU reference (`include/ll/mlp_ref.hpp` pattern). Tolerances stated explicitly.
3. **Report percentiles** (p50/p99/p99.9/max) via `ll::LatencyRecorder`, never just averages.
4. **Avoid coordinated omission:** timestamp at the *scheduled* send time (see `apps/spsc_latency.cpp`).
5. **No heap allocation, locks, syscalls or exceptions on hot paths.** Pre-allocate, use arenas/ring buffers.
6. Pad/align shared hot state with `ll::kCacheLine`. Keep producer-written and consumer-written fields on separate lines.
7. Host code in `include/ll/` stays header-only and compiles without CUDA. CUDA code lives in `cuda/`.
8. Keep each phase in its own CMake target so variants can be compared side by side.

## Build & test
```bash
cmake -S . -B build -G Ninja                 # CUDA targets auto-enabled if nvcc is found
cmake --build build
ctest --test-dir build --output-on-failure
cmake -S . -B build-tsan -G Ninja -DLL_SANITIZE=thread && cmake --build build-tsan   # race checks
```
CPU-only: `-DLL_ENABLE_CUDA=OFF`. Different GPU: `-DCMAKE_CUDA_ARCHITECTURES=<arch>`.

## Layout
- `include/ll/`: `platform.hpp` (clock, pinning, cache line), `latency_recorder.hpp` (histogram), `spsc_queue.hpp`, `mlp_ref.hpp` (reference model)
- `apps/`: `spsc_latency` (queue latency), `cpu_mlp_baseline`
- `cuda/`: `gpu_probe` (launch/copy overheads), `gpu_mlp_baseline` (CPU vs GPU curve). **Written without nvcc access: expect small first-build fixes.**
- `bench/`: Google Benchmark micro-benchmarks. `tests/`: correctness tests.

## Roadmap (tick off as done)
- [ ] **Phase 0:** build on dev machine, run `gpu_probe` and `gpu_mlp_baseline`, record the WSL2 baseline
- [ ] **Phase 1:** ITCH 5.0 parser (mmap, zero-copy) + flat-array limit order book; replay sample Nasdaq ITCH data; report msgs/s/core and cache misses (`perf stat`)
- [ ] **Phase 2:** pinned memory + CUDA graphs tuning; fused feature+inference kernel across many symbols; FP16 / TensorRT variants; latency-vs-batch curves
- [ ] **Phase 3:** full pipeline (replay -> book -> features -> GPU -> signal) with SPSC queues, thread pinning, per-stage histograms; rerun on native Linux with isolated cores
- [ ] **Stretch:** persistent-kernel inference with host-GPU mailbox (needs headless GPU or iGPU for display), FP8/FP4 on Blackwell, hand-written GPU GBDT inference vs XGBoost/FIL

## Style
C++20, `-Wall -Wextra -Wpedantic -Wshadow` clean. Small, measurable commits. When adding a target, add a one-line description to this file and the roadmap.
