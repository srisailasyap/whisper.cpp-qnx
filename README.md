# whisper.cpp-qnx

Build files to cross-compile [whisper.cpp](https://github.com/ggml-org/whisper.cpp) (OpenAI Whisper C/C++ port) for the **QNX Neutrino RTOS 8.0** on `aarch64le` and `x86_64`.

## Upstream version

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
