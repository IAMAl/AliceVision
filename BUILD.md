# Building this AliceVision fork (Linux, from source, reproducible)

This branch (`local-build-fixes`) builds AliceVision **3.4.0** (develop, base `fd283da`) on
**Ubuntu 22.04** with **gcc-12** and **CUDA 12.4** (NVIDIA RTX 4090, compute capability
`sm_89`) via the embedded-dependency superbuild. It contains three fixes that the upstream
superbuild needs in this environment:

1. `cmake/Helpers.cmake` — full clone for commit-SHA-pinned git deps (LEMON)
2. `cmake/deps/swig.cmake` — `SWIG_EXECUTABLE` path (`bin/`, not `bin-deps/`) so the
   `pyalicevision` bindings build (needed by ~23 Meshroom node descriptors)
3. `nonFree/sift/CMakeLists.txt` — pass `-mavx` to the VLFeat AVX kernels on GCC/Clang

> These build the **AliceVision C++ library + binaries** only. Wiring it into Meshroom
> (and building QtAliceVision for the 2D/3D viewers) is separate — see the `run-meshroom-dev.sh`
> launcher in the Meshroom fork.

## 1. System prerequisites (apt)

```bash
sudo apt-get update && sudo apt-get install -y --no-install-recommends \
    build-essential gcc-12 g++-12 gfortran-12 \
    automake autoconf libtool bison texinfo \
    nasm yasm pkg-config \
    libpcre2-dev libssl-dev libxerces-c-dev libyaml-cpp-dev \
    curl wget unzip file git
```

The superbuild builds the heavy C++ deps from source (Boost, OpenImageIO ≥3.0, OpenEXR,
FFmpeg, OpenColorIO, OpenCV, Ceres, Geogram, Alembic, USD, ONNXRuntime, PopSift, CCTag,
AprilTag, …), so they are **not** apt packages.

## 2. CMake 3.31 (important)

AliceVision's top-level needs CMake ≥ 3.30, but the bundled deps use pre-3.5
`cmake_minimum_required`, which **CMake 4.x rejects**. Use a 3.31.x in between (no root):

```bash
pip install --user "cmake>=3.31,<3.32"
hash -r && cmake --version   # -> 3.31.x  (ensure ~/.local/bin is first on PATH)
```

## 3. CUDA

CUDA ≥ 11 is required for the GPU nodes (DepthMap, PopSift). This build used **CUDA 12.4**
at `/usr/local/cuda` (driver 595 supports up to 13.2). CUDA 12.4 supports gcc ≤ 13, so
gcc-12 is fine. To build CPU-only instead, pass `-DALICEVISION_USE_CUDA=OFF`.

## 4. Build

```bash
git clone https://github.com/IAMAl/AliceVision.git
cd AliceVision
git checkout local-build-fixes

export PATH="$HOME/.local/bin:/usr/local/cuda/bin:$PATH"   # cmake 3.31 + nvcc
export CC=gcc-12 CXX=g++-12

cmake -B build -S . \
  -DALICEVISION_BUILD_DEPENDENCIES=ON \
  -DCMAKE_INSTALL_PREFIX="$PWD/install" \
  -DALICEVISION_USE_CUDA=ON \
  -DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda \
  -DCMAKE_CUDA_ARCHITECTURES=89 \
  -DCMAKE_BUILD_TYPE=Release

cmake --build build -j"$(nproc)"     # ~1–3 h on 32 cores; downloads + builds all deps
```

`CMAKE_CUDA_ARCHITECTURES=89` targets the RTX 4090 — change to your GPU's compute capability
(e.g. `86` for RTX 30xx, `75` for RTX 20xx).

## 5. Result

Installs to `./install`:
- `bin/` — 236 `aliceVision_*` executables (CameraInit, FeatureExtraction, DepthMap, …)
- `lib/` — AliceVision + all dependency shared libs
- `lib/python/pyalicevision/` — SWIG Python bindings
- `share/meshroom/` — Meshroom node descriptors (`items=` API) + pipeline `.mg` templates
- `share/aliceVision/` — `cameraSensors.db`, OCIO config, models

Quick check (binaries need the install libs on the path):
```bash
LD_LIBRARY_PATH=install/lib:/usr/local/cuda/lib64 ALICEVISION_ROOT=install \
  install/bin/aliceVision_cameraInit --help
```

## Notes / gotchas
- The bundled binaries' `RUNPATH` is `$ORIGIN/../lib`; in practice set
  `LD_LIBRARY_PATH=install/lib` when running them (and `/usr/local/cuda/lib64` for GPU nodes).
- The build needs ~30–50 GB of disk for source + objects.
- `spirv-opt not found` warnings (if any tool reaches shader baking) are harmless.
- Resume after a failed dep: just re-run the `cmake --build` step — completed deps are cached
  via ExternalProject stamps.
