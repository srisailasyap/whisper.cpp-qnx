# whisper.cpp port to QNX

This is a port of GGML's [whisper.cpp](https://github.com/ggml-org/whisper.cpp) <img src="https://avatars.githubusercontent.com/u/134263123?s=48&v=4" width=24 /> project to **QNX Neutrino RTOS 8.0**. Currently only supports CPU-based ggml. With whisper.cpp you can run OpenAI's Whisper speech-to-text models fully on-device — no GPU, no internet — on QNX targets including automotive, robotics, and industrial platforms.

Supported models: `tiny`, `base`, `small`, `medium`, `large-v1/v2/v3`, plus all English-only `.en` variants and distilled models (`distil-whisper`).

The latest release was tested with **whisper.cpp `v1.8.4`** on a Raspberry Pi 5 (QNX 8.0, `aarch64le`) and `x86_64` QNX targets, using the `ggml-tiny.en` and `ggml-base.en` models.

QNX-specific changes (committed on top of upstream v1.8.4):
- ggml CPU detection via QNX `syspage` (SIMD feature probing)
- `aarch64le` / `nto-aarch64-le` recognized by the ggml CMake regex
- `getcwd()` fallback in `ggml-backend-reg.cpp` (QNX lacks `std::filesystem`)
- `mmap`-based Whisper model loader for faster load times on QNX filesystems

| Item | Value |
|---|---|
| whisper.cpp tag | **`v1.8.4`** |
| Source fork (QNX patches applied) | [srisailasyap/whisper.cpp @ qnx-v1.8.4](https://github.com/srisailasyap/whisper.cpp/tree/qnx-v1.8.4) |
| QNX SDP | 8.0 |
| Tested targets | Raspberry Pi 5 (`nto-aarch64-le`), `nto-x86_64-o` |

> **NOTE:** QNX ports are only supported from a Linux host operating system.

Use `$(nproc)` instead of `4` after `JLEVEL=` and `-j` to use the maximum number of cores. 32GB of RAM is recommended when using `JLEVEL=$(nproc)` or `-j$(nproc)`.

## Build in a Docker container

Pre-requisite: [Install Docker on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

```bash
# Create a workspace
mkdir -p ~/qnx_workspace && cd ~/qnx_workspace

# Clone the QNX build-files repo (provides docker scripts)
git clone https://github.com/qnx-ports/build-files.git

# Build the Docker image and create a container
cd build-files/docker
./docker-build-qnx-image.sh
./docker-create-container.sh

# Inside the Docker container:
source ~/qnx800/qnxsdp-env.sh

# Clone this port and the QNX-patched whisper.cpp fork
cd ~/qnx_workspace
git clone https://github.com/srisailasyap/whisper.cpp-qnx.git
git clone -b qnx-v1.8.4 https://github.com/srisailasyap/whisper.cpp.git

# Build whisper.cpp
QNX_PROJECT_ROOT="$(pwd)/whisper.cpp" make -C whisper.cpp-qnx install -j4
```

## Build natively on a host

```bash
mkdir -p ~/qnx_workspace && cd ~/qnx_workspace

# Clone this port and the QNX-patched whisper.cpp fork
git clone https://github.com/srisailasyap/whisper.cpp-qnx.git
git clone -b qnx-v1.8.4 https://github.com/srisailasyap/whisper.cpp.git

# Source the QNX SDP environment
source ~/qnx800/qnxsdp-env.sh

# Build
QNX_PROJECT_ROOT="$(pwd)/whisper.cpp" make -C whisper.cpp-qnx install -j4
```

## Download a model

whisper.cpp requires a GGML-format Whisper model:

```bash
cd ~/qnx_workspace/whisper.cpp/models
./download-ggml-model.sh tiny.en
```

## Run on the target

Transfer the binaries, libraries, model, and test audio to the QNX target:

```bash
TARGET_HOST=<target-ip-address-or-hostname>

# Binaries
scp $QNX_TARGET/aarch64le/usr/local/bin/whisper-* qnxuser@$TARGET_HOST:/data/home/qnxuser/

# Shared libraries
scp $QNX_TARGET/aarch64le/usr/local/lib/libwhisper* qnxuser@$TARGET_HOST:/data/home/qnxuser/
scp $QNX_TARGET/aarch64le/usr/local/lib/libggml*    qnxuser@$TARGET_HOST:/data/home/qnxuser/

# Model + test audio
scp ~/qnx_workspace/whisper.cpp/models/ggml-tiny.en.bin qnxuser@$TARGET_HOST:/data/home/qnxuser/
scp ~/qnx_workspace/whisper.cpp/samples/jfk.wav         qnxuser@$TARGET_HOST:/data/home/qnxuser/
```

Run on the target:

```bash
ssh qnxuser@$TARGET_HOST

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/data/home/qnxuser
export GGML_BACKEND_PATH=/data/home/qnxuser

cd /data/home/qnxuser

# Speech-to-text
./whisper-cli -m ggml-tiny.en.bin jfk.wav

# Benchmark
./whisper-bench -m ggml-tiny.en.bin
```

## Supported architectures

- `aarch64le`
- `x86_64`

## Building a different whisper.cpp version

To rebuild against a different upstream tag:

1. Check out the desired tag in a fresh whisper.cpp clone.
2. Apply `whisper.cpp.patch` from this repo (`git apply whisper.cpp.patch`) and resolve any conflicts.
3. Regenerate the patch: `git diff > whisper.cpp.patch`.
4. Update the tag reference in this README.
