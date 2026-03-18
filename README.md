# aapt2_android
Cross-compiled [aapt2](https://developer.android.com/tools/aapt2) binaries for
Android (aarch64), built in CI and suitable for use in [Termux](https://termux.dev).

## Build approaches

Two independent GitHub Actions workflows are provided:

| Workflow | Source | File |
|---|---|---|
| `build-aapt2-aarch64.yml` | [Lzhiyong/sdk-tools](https://github.com/Lzhiyong/sdk-tools) (third-party, CMake-ready) | `.github/workflows/build-aapt2-aarch64.yml` |
| `build-aapt2-aarch64-aosp.yml` | Official AOSP (`android.googlesource.com`) via `repo` | `.github/workflows/build-aapt2-aarch64-aosp.yml` |

Both workflows produce a Termux-compatible `aapt2` binary for `arm64-v8a`
(Android API ≥ 26) and upload it as a build artifact.

## Building from official AOSP sources with `repo`

The AOSP workflow (`build-aapt2-aarch64-aosp.yml`) uses Google's
[`repo`](https://gerrit.googlesource.com/git-repo) tool to fetch the exact
AOSP sources needed and nothing more:

```
manifests/aosp-aapt2.xml   # repo manifest — declares which AOSP projects to sync
cmake/CMakeLists.txt       # CMake build configuration for the AOSP source layout
```

### Synced AOSP projects (android-14.0.0_r1)

| Project | Local path | Contents used |
|---|---|---|
| `platform/frameworks/base` | `frameworks/base` | `tools/aapt2/`, `libs/androidfw/` |
| `platform/system/libbase` | `system/libbase` | string/file utilities |
| `platform/system/core` | `system/core` | `libcutils/`, `libutils/`, `libziparchive/` |
| `platform/system/logging` | `system/logging` | `liblog/` |
| `platform/external/protobuf` | `external/protobuf` | protobuf-cpp-full runtime |
| `platform/external/libpng` | `external/libpng` | PNG support |
| `platform/external/expat` | `external/expat` | XML parsing |
| `platform/external/zlib` | `external/zlib` | zlib (fallback if NDK libz absent) |

Sparse checkout is used for `frameworks/base` and `system/core` so that only
the sub-directories actually required are downloaded, keeping CI fast.

### Running locally

```bash
# 1. Create a workspace and initialize repo
mkdir aosp-aapt2-ws && cd aosp-aapt2-ws
repo init -u https://github.com/xingguangcuican6666/aapt2_android \
          -m manifests/aosp-aapt2.xml \
          --partial-clone --clone-filter=blob:none --depth=1

# 2. Sync sources
repo sync -c --no-tags --no-clone-bundle -j$(nproc)

# 3. Cross-compile with NDK (adjust NDK path as needed)
NDK=$HOME/Android/Sdk/ndk/26.3.11579264
cmake -B build -S /path/to/aapt2_android/cmake \
      -DAOSP_ROOT="$PWD" \
      -DCMAKE_TOOLCHAIN_FILE="$NDK/build/cmake/android.toolchain.cmake" \
      -DANDROID_ABI=arm64-v8a \
      -DANDROID_PLATFORM=android-26 \
      -DANDROID_STL=c++_static \
      -DCMAKE_BUILD_TYPE=Release \
      -G Ninja
cmake --build build --target aapt2
```
