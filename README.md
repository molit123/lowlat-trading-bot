# lowlat-gpu

Starter kit for a C++ / CUDA project on **low-latency systems design and GPU inference**, built around one question: *where does a GPU actually beat a CPU for small trading-style models, once you pay for launch and transfer overhead?*

Status: **skeleton.** The CPU side is built and tested; the CUDA files are untested because they were written without `nvcc` available. Expect small compile fixes on first build.

## Quick start
```bash
# See docs/SETUP.md first (WSL2 + CUDA 12.8+, or native Ubuntu).
cmake -S . -B build -G Ninja        # add -DLL_ENABLE_CUDA=OFF for CPU-only
cmake --build build
ctest --test-dir build --output-on-failure

./build/spsc_latency --prod-cpu 2 --cons-cpu 3 --interval-ns 1000   # queue latency percentiles
./build/cpu_mlp_baseline                                            # CPU inference vs batch size
./build/micro_bench                                                 # clock/histogram/queue/MLP micro-benchmarks
./build/gpu_probe                                                   # launch + copy overheads   (CUDA)
./build/gpu_mlp_baseline --csv results.csv                          # CPU vs GPU vs batch size  (CUDA)
```

## What's here
| Piece | File | Purpose |
|---|---|---|
| Latency histogram | `include/ll/latency_recorder.hpp` | Allocation-free, ~1.6% precision, percentiles + CSV |
| SPSC queue | `include/ll/spsc_queue.hpp` | Cache-line padded, cached indices, acquire/release |
| Reference model | `include/ll/mlp_ref.hpp` | 64-128-128-1 MLP; ground truth for every fast path |
| Overhead probe | `cuda/gpu_probe.cu` | Cost of kernel launch, graph launch, pinned vs pageable copies |
| CPU vs GPU curve | `cuda/gpu_mlp_baseline.cu` | The central experiment, with correctness check |

## Results template
| Environment | Batch | CPU p50 / p99 | GPU stream p50 / p99 | GPU graph p50 / p99 | Winner |
|---|---|---|---|---|---|
| WSL2 | 1 | | | | |
| native Ubuntu (isolated cores) | 1 | | | | |

See `CLAUDE.md` for ground rules and the roadmap.
