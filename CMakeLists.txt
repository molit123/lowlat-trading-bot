cmake_minimum_required(VERSION 3.24)

# RTX 5070 Ti = Blackwell, compute capability 12.0 (sm_120). Needs CUDA toolkit >= 12.8.
# Must be set before the CUDA language is enabled. Override with -DCMAKE_CUDA_ARCHITECTURES=...
if(NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
  set(CMAKE_CUDA_ARCHITECTURES 120 CACHE STRING "CUDA architectures (120 = RTX 50xx)")
endif()

project(lowlat_gpu VERSION 0.1.0 LANGUAGES CXX)

option(LL_ENABLE_CUDA "Build CUDA targets if a CUDA compiler is found" ON)
option(LL_BUILD_BENCH "Build Google Benchmark micro-benchmarks" ON)
option(LL_BUILD_TESTS "Build correctness tests" ON)
option(LL_NATIVE "Compile CPU code with -march=native (-mcpu=native on ARM)" ON)
set(LL_SANITIZE "none" CACHE STRING "none | address | thread | undefined")
set(LL_CACHELINE "" CACHE STRING "Override cache-line size in bytes (default: 64 x86, 128 aarch64)")

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "" FORCE)
endif()
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)  # GNU extensions: asm volatile, __restrict, etc.

find_package(Threads REQUIRED)

# ---------------------------------------------------------------- CUDA (optional)
set(LL_HAVE_CUDA OFF)
if(LL_ENABLE_CUDA)
  include(CheckLanguage)
  check_language(CUDA)
  if(CMAKE_CUDA_COMPILER)
    enable_language(CUDA)
    set(CMAKE_CUDA_STANDARD 20)
    set(CMAKE_CUDA_STANDARD_REQUIRED ON)
    set(LL_HAVE_CUDA ON)
    if("120" IN_LIST CMAKE_CUDA_ARCHITECTURES AND CMAKE_CUDA_COMPILER_VERSION VERSION_LESS 12.8)
      message(WARNING "nvcc ${CMAKE_CUDA_COMPILER_VERSION} cannot target sm_120 (RTX 50xx needs >= 12.8). "
                      "Upgrade the toolkit or pass -DCMAKE_CUDA_ARCHITECTURES=<your arch>.")
    endif()
    message(STATUS "CUDA ${CMAKE_CUDA_COMPILER_VERSION}, architectures: ${CMAKE_CUDA_ARCHITECTURES}")
  else()
    message(STATUS "No CUDA compiler found -> CPU-only build. "
                   "(If it's installed, try -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc)")
  endif()
endif()

# ---------------------------------------------------------------- shared options
add_library(ll_opts INTERFACE)
target_compile_options(ll_opts INTERFACE
  $<$<COMPILE_LANGUAGE:CXX>:-Wall -Wextra -Wpedantic -Wshadow>
  $<$<COMPILE_LANGUAGE:CXX>:-fno-omit-frame-pointer>   # readable perf / Nsight stacks
  $<$<COMPILE_LANGUAGE:CUDA>:-Xcompiler=-Wall,-Wextra>
  $<$<COMPILE_LANGUAGE:CUDA>:-lineinfo>)               # source correlation in Nsight Compute

if(LL_NATIVE)
  include(CheckCXXCompilerFlag)
  check_cxx_compiler_flag(-march=native LL_HAS_MARCH_NATIVE)
  if(LL_HAS_MARCH_NATIVE)
    target_compile_options(ll_opts INTERFACE $<$<COMPILE_LANGUAGE:CXX>:-march=native>)
  else()
    check_cxx_compiler_flag(-mcpu=native LL_HAS_MCPU_NATIVE)
    if(LL_HAS_MCPU_NATIVE)
      target_compile_options(ll_opts INTERFACE $<$<COMPILE_LANGUAGE:CXX>:-mcpu=native>)
    endif()
  endif()
endif()

if(NOT LL_SANITIZE STREQUAL "none")
  target_compile_options(ll_opts INTERFACE $<$<COMPILE_LANGUAGE:CXX>:-fsanitize=${LL_SANITIZE} -g>)
  target_link_options(ll_opts INTERFACE -fsanitize=${LL_SANITIZE})
endif()

if(NOT LL_CACHELINE STREQUAL "")
  target_compile_definitions(ll_opts INTERFACE LL_CACHELINE=${LL_CACHELINE})
endif()

# Header-only core library: LatencyRecorder, SpscQueue, platform helpers, MLP reference.
add_library(ll INTERFACE)
target_include_directories(ll INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_link_libraries(ll INTERFACE ll_opts Threads::Threads)

# ---------------------------------------------------------------- CPU apps
add_executable(spsc_latency apps/spsc_latency.cpp)
target_link_libraries(spsc_latency PRIVATE ll)

add_executable(cpu_mlp_baseline apps/cpu_mlp_baseline.cpp)
target_link_libraries(cpu_mlp_baseline PRIVATE ll)

# ---------------------------------------------------------------- CUDA apps
if(LL_HAVE_CUDA)
  add_executable(gpu_probe cuda/gpu_probe.cu)
  target_link_libraries(gpu_probe PRIVATE ll)

  add_executable(gpu_mlp_baseline cuda/gpu_mlp_baseline.cu)
  target_link_libraries(gpu_mlp_baseline PRIVATE ll)
endif()

# ---------------------------------------------------------------- tests
if(LL_BUILD_TESTS)
  enable_testing()
  add_executable(test_core tests/test_core.cpp)
  target_link_libraries(test_core PRIVATE ll)
  add_test(NAME core COMMAND test_core)
endif()

# ---------------------------------------------------------------- micro-benchmarks
if(LL_BUILD_BENCH)
  find_package(benchmark QUIET)
  if(NOT benchmark_FOUND)
    include(FetchContent)
    set(BENCHMARK_ENABLE_TESTING OFF CACHE BOOL "" FORCE)
    set(BENCHMARK_ENABLE_GTEST_TESTS OFF CACHE BOOL "" FORCE)
    set(BENCHMARK_ENABLE_INSTALL OFF CACHE BOOL "" FORCE)
    FetchContent_Declare(benchmark
      GIT_REPOSITORY https://github.com/google/benchmark.git
      GIT_TAG v1.9.1)
    FetchContent_MakeAvailable(benchmark)
  endif()
  add_executable(micro_bench bench/micro_bench.cpp)
  target_link_libraries(micro_bench PRIVATE ll benchmark::benchmark)
endif()

# ---------------------------------------------------------------- roadmap targets (add here)
# Phase 1: apps/itch_replay.cpp + include/ll/order_book.hpp   (ITCH 5.0 parser + flat-array LOB)
# Phase 2: cuda/fused_features_infer.cu                       (fused feature + inference kernel)
# Phase 3: apps/pipeline.cpp                                  (replay -> book -> features -> GPU -> signal)
