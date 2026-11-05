# Building and Installing vLLM from Source on Ubuntu (CPU Only)

Source: https://docs.vllm.ai/en/latest/getting_started/installation/cpu/#build-wheel-from-source
## 1. Overview

This document describes how to set up a fresh Ubuntu machine, install the required system dependencies, download the vLLM source code, build vLLM from source, and install it locally.

The instructions follow the CPU-only build configuration used by the project's `docker/Dockerfile.cpu`.

The procedure does not require Docker, CUDA, NVIDIA drivers, or a GPU.

### Supported configurations

| Component | Configuration |
|---|---|
| Operating system | Ubuntu 22.04 LTS |
| CPU architecture | x86_64 (AMD64) and ARM64 (AArch64) |
| Python | 3.12 |
| C/C++ compiler | GCC 15 / G++ 15 |
| Build system | CMake, Make, Ninja |
| Python package manager | uv |
| Rust toolchain | Installed using rustup |
| GNU Binutils | 2.47 |
| PyTorch | CPU-only |
| vLLM | Built from source |

**Important:** Ubuntu 22.04 is the reference operating system because the Dockerfile uses `ubuntu:22.04` as its base image. Other Ubuntu versions may work, but package availability, compiler compatibility, and repository configuration should be verified separately.

---

## 2. Installation directory structure

The following directory structure is used throughout this guide.

All user-specific installation directories are derived from `$HOME`.

| Component | Installation path |
|---|---|
| vLLM source code | `$HOME/src/vllm` |
| Python virtual environment | `$HOME/.venvs/vllm` |
| uv Python installation | `$HOME/.local/share/uv/python` |
| Rust toolchain | `$HOME/.cargo` |
| Rust binaries | `$HOME/.cargo/bin` |
| GNU Binutils | `$HOME/.local/opt/binutils-2.47` |
| Compiler cache | `$HOME/.cache/ccache` |
| uv cache | `$HOME/.cache/uv` |
| Build artifacts | `$HOME/src/vllm/dist` |

No commands in this document require a hardcoded username.

---

## 3. Check the Ubuntu installation and CPU architecture

Before installing dependencies, verify the operating system and CPU architecture.

```bash
cat /etc/os-release
uname -m
```

Expected architecture values:

- `x86_64`: Intel/AMD 64-bit CPU.
- `aarch64`: ARM 64-bit CPU.

Check available CPU cores and memory:

```bash
nproc
lscpu
free -h
df -h "$HOME"
```

Ensure sufficient free disk space for the source code, Python packages, compiler intermediates, and build artifacts.

A build can consume considerably more disk space and memory than the final installed package.

---

## 4. Install Ubuntu system dependencies

Update the Ubuntu package repositories:

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

Install the system packages required for building vLLM.

```bash
sudo apt-get install -y \
    ca-certificates \
    curl \
    wget \
    git \
    jq \
    unzip \
    zip \
    tar \
    gzip \
    xz-utils \
    bzip2 \
    software-properties-common \
    apt-transport-https \
    gnupg \
    lsb-release \
    sudo \
    build-essential \
    gcc \
    g++ \
    make \
    cmake \
    ninja-build \
    pkg-config \
    autoconf \
    automake \
    libtool \
    bison \
    flex \
    texinfo \
    zlib1g-dev \
    libnuma-dev \
    python3 \
    python3-dev \
    python3-pip \
    python3-venv \
    ffmpeg \
    libsm6 \
    libxext6 \
    libgl1 \
    libtcmalloc-minimal4 \
    numactl \
    htop \
    lsof \
    procps \
    sysstat \
    pciutils \
    util-linux \
    iputils-ping \
    net-tools \
    dnsutils \
    iproute2 \
    netcat-openbsd \
    gdb \
    strace \
    file \
    tree \
    vim \
    less \
    tmux \
    libgomp1 \
    libomp-dev \
    ccache
```

### Package descriptions

| Package | Purpose |
|---|---|
| build-essential | Essential C/C++ compilation tools |
| gcc / g++ | C and C++ compilers |
| cmake | CMake build configuration |
| ninja-build | Fast build system |
| pkg-config | Library and compiler flag discovery |
| zlib1g-dev | Compression development library |
| libnuma-dev | NUMA development library |
| libtcmalloc-minimal4 | Memory allocation optimization |
| ffmpeg | Multimedia dependency |
| jq | JSON processing utility |
| ccache | C/C++ compilation cache |
| numactl | NUMA management |
| libgomp1 | GNU OpenMP runtime |
| libomp-dev | LLVM OpenMP development files |

The additional utilities (`htop`, `jq`, `gdb`, `strace`, `tree`, etc.) are intended to make development and troubleshooting easier. They are not all direct vLLM compilation dependencies.

---

## 5. Install GCC 15 and G++ 15

The CPU Dockerfile uses GCC 15 and G++ 15.

Add the Ubuntu Toolchain PPA:

```bash
sudo add-apt-repository -y ppa:ubuntu-toolchain-r/test
sudo apt-get update
```

Install GCC 15:

```bash
sudo apt-get install -y gcc-15 g++-15
```

Configure GCC 15 as the default compiler:

```bash
sudo update-alternatives \
    --install /usr/bin/gcc gcc /usr/bin/gcc-15 10 \
    --slave /usr/bin/g++ g++ /usr/bin/g++-15
```

Verify the compiler:

```bash
gcc --version
g++ --version
```

Expected:

```text
gcc (Ubuntu ...) 15.x
g++ (Ubuntu ...) 15.x
```

Set the compiler environment variables:

```bash
export CC=/usr/bin/gcc-15
export CXX=/usr/bin/g++-15
```

**Note:** If the Toolchain PPA does not provide GCC 15 for your Ubuntu release or architecture, do not force-install packages built for another Ubuntu release. Use a compatible compiler installation and verify compatibility with the vLLM build configuration.

---

## 6. Install GNU Binutils 2.47

The Dockerfile builds GNU Binutils 2.47 to provide a newer assembler for AMX-FP8 instruction support.

Build and install Binutils locally rather than modifying the system-wide Binutils installation.

Set the installation directory:

```bash
export BINUTILS_VERSION=2.47
export BINUTILS_PREFIX="$HOME/.local/opt/binutils-$BINUTILS_VERSION"
```

Download and extract the source:

```bash
cd "$HOME"

wget "https://sourceware.org/pub/binutils/releases/binutils-${BINUTILS_VERSION}.tar.xz"

tar -xf "binutils-${BINUTILS_VERSION}.tar.xz"
```

Create the build directory:

```bash
mkdir -p "$HOME/build/binutils"
cd "$HOME/build/binutils"
```

Configure:

```bash
"$HOME/binutils-${BINUTILS_VERSION}/configure" \
    --prefix="$BINUTILS_PREFIX" \
    --disable-nls \
    --disable-werror
```

Compile:

```bash
make -j"$(nproc)"
```

Install:

```bash
make install
```

Clean up downloaded source files:

```bash
rm -rf \
    "$HOME/build/binutils" \
    "$HOME/binutils-${BINUTILS_VERSION}" \
    "$HOME/binutils-${BINUTILS_VERSION}.tar.xz"
```

Configure the compiler environment:

```bash
export PATH="$BINUTILS_PREFIX/bin:$PATH"
export COMPILER_PATH="$BINUTILS_PREFIX/bin"
```

Verify:

```bash
as --version
ld --version
```

The assembler should report GNU Binutils 2.47.

**Note:** This follows the Dockerfile's Binutils build stage. It is particularly relevant to the x86 AMX-FP8 build configuration. If AMX-FP8 is disabled and the corresponding instructions are not required, this custom Binutils build may be unnecessary.

---

## 7. Install uv

vLLM uses `uv` for Python environment management and dependency installation.

Install uv:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Add uv to the current shell's PATH:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Verify:

```bash
uv --version
```

To make the PATH configuration persistent:

```bash
grep -qxF 'export PATH="$HOME/.local/bin:$PATH"' "$HOME/.bashrc" \
    || echo 'export PATH="$HOME/.local/bin:$PATH"' >> "$HOME/.bashrc"
```

Reload the shell configuration:

```bash
source "$HOME/.bashrc"
```

---

## 8. Install Python 3.12 using uv

The Dockerfile uses Python 3.12 by default.

Install Python 3.12:

```bash
uv python install 3.12
```

Create a dedicated Python virtual environment:

```bash
mkdir -p "$HOME/.venvs"

uv venv \
    --python 3.12 \
    "$HOME/.venvs/vllm"
```

Activate the environment:

```bash
source "$HOME/.venvs/vllm/bin/activate"
```

Verify:

```bash
python --version
which python
uv --version
```

The Python executable should resolve to:

```text
$HOME/.venvs/vllm/bin/python
```

All subsequent Python dependencies will be installed into this virtual environment.

---

## 9. Install Rust build dependencies

The Dockerfile builds the Rust frontend (`vllm-rs`) separately from the main C++/Python build.

The repository provides the Rust toolchain configuration through:

```text
rust-toolchain.toml
```

The build script:

```text
tools/build_rust.sh
```

handles Rust toolchain installation using rustup.

Install the Rust build dependencies:

```bash
cd "$HOME/src/vllm"

uv pip install -r requirements/build/rust.txt
```

The Rust build script will install the toolchain specified by the repository.

Verify Rust after running the build script:

```bash
rustc --version
cargo --version
rustup --version
```

If you want to install rustup explicitly before running the build script:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y

source "$HOME/.cargo/env"
```

Rust binaries are installed under:

```text
$HOME/.cargo/bin
```

Ensure this directory is in PATH:

```bash
export PATH="$HOME/.cargo/bin:$PATH"
```

---

## 10. Download the vLLM source code

Create the source directory:

```bash
mkdir -p "$HOME/src"
cd "$HOME/src"
```

Clone the vLLM repository:

```bash
git clone https://github.com/vllm-project/vllm.git
```

Navigate to the source directory:

```bash
cd "$HOME/src/vllm"
```

To build a specific release or branch, check out the required Git reference.

For example:

```bash
git checkout v0.33.0
```

Alternatively, check out your required development branch:

```bash
git checkout <branch-name>
```

If building from a fork, replace the repository URL with the appropriate fork URL.

Verify the source tree:

```bash
git status
git log -1 --oneline
```

**Important:** The repository's source code, requirements files, Rust configuration, and build scripts must all come from the same Git revision.

---

## 11. Configure the CPU build environment

Navigate to the repository:

```bash
cd "$HOME/src/vllm"
```

Set the CPU-only build configuration:

```bash
export VLLM_TARGET_DEVICE=cpu

export CC=/usr/bin/gcc-15
export CXX=/usr/bin/g++-15

export CCACHE_DIR="$HOME/.cache/ccache"
export CMAKE_CXX_COMPILER_LAUNCHER=ccache

export PATH="$HOME/.local/bin:$HOME/.cargo/bin:$BINUTILS_PREFIX/bin:$PATH"
```

Create the compiler cache directory:

```bash
mkdir -p "$CCACHE_DIR"
```

Set the cache and build directories:

```bash
mkdir -p "$HOME/.cache/uv"
mkdir -p "$HOME/.cache/ccache"
```

Configure the build environment:

```bash
export UV_HTTP_TIMEOUT=500
export UV_INDEX_STRATEGY=unsafe-best-match
export UV_LINK_MODE=copy
export UV_COMPILE_BYTECODE=1
```

For a normal build with sufficient memory:

```bash
export MAX_JOBS="$(nproc)"
export CARGO_BUILD_JOBS=4
```

For systems with limited memory, see Section 15 before proceeding.

---

## 12. Install CPU-only PyTorch and vLLM dependencies

Install the CPU runtime dependencies:

```bash
cd "$HOME/src/vllm"

uv pip install --upgrade pip

uv pip install \
    -r requirements/cpu.txt \
    --torch-backend cpu
```

This installs the CPU-only PyTorch dependency configuration specified by the repository.

Next, install the CPU build dependencies:

```bash
uv pip install \
    -r requirements/build/cpu.txt \
    --torch-backend cpu
```

The requirements files are maintained by the vLLM repository. Use the versions from the checked-out revision rather than manually replacing individual package versions.

Verify PyTorch:

```bash
python -c "import torch; print('PyTorch:', torch.__version__); print('CUDA available:', torch.cuda.is_available()); print('CPU threads:', torch.get_num_threads())"
```

Expected:

```text
CUDA available: False
```

The exact PyTorch version depends on the checked-out vLLM revision.

---

## 13. Build the Rust frontend

The Dockerfile uses a dedicated Rust build stage to compile the `vllm-rs` frontend before the main vLLM wheel build.

For a native local build, execute the repository's Rust build script:

```bash
cd "$HOME/src/vllm"

bash tools/build_rust.sh
```

This script uses the Rust configuration and build dependencies provided by the repository.

Verify that the Rust artifacts were generated:

```bash
ls -lah vllm/vllm-rs
ls -lah vllm/_rust_*.so
```

The exact artifacts depend on the repository revision.

If the Rust build fails, inspect the output before continuing with the Python/C++ build.

---

## 14. Build vLLM from source

Once the system dependencies, Python dependencies, CPU PyTorch, and Rust frontend are installed, build the vLLM wheel.

Navigate to the source directory:

```bash
cd "$HOME/src/vllm"
```

Configure the build:

```bash
export VLLM_TARGET_DEVICE=cpu
export CC=/usr/bin/gcc-15
export CXX=/usr/bin/g++-15

export CMAKE_ARGS="-DENABLE_AMX_FP8=1"
```

Build the wheel:

```bash
python setup.py bdist_wheel \
    --dist-dir=dist \
    --py-limited-api=cp38
```

This follows the wheel-building invocation used in the CPU Dockerfile.

The build may take a considerable amount of time depending on CPU architecture, available RAM, CPU core count, and compiler cache state.

The resulting wheel will be available under:

```text
$HOME/src/vllm/dist/
```

List the generated wheel:

```bash
ls -lh "$HOME/src/vllm/dist/"
```

**AMX-FP8 note:** The example above follows the Dockerfile's default `ENABLE_AMX_FP8=1`. For systems that do not require AMX-FP8, or where the corresponding toolchain/instructions are unavailable, check the selected revision's supported build configuration before disabling or changing this option.

---

## 15. Building on systems with limited RAM

C++ compilation, PyTorch-related dependency builds, CMake, and Rust compilation can consume substantial memory.

On machines with limited RAM, allowing every build process to run concurrently can cause:

- Excessive memory usage.
- System swapping.
- Out-of-memory (OOM) termination.
- Compiler processes being killed.
- Incomplete or failed builds.

The build parallelism should be reduced according to the available system memory.

### 15.1 Control uv concurrent builds

`uv` supports the `UV_CONCURRENT_BUILDS` environment variable.

This controls the maximum number of source distributions that uv builds concurrently.

For machines with limited RAM, set:

```bash
export UV_CONCURRENT_BUILDS=1
```

This limits uv to one concurrent source build.

For systems with more memory:

```bash
export UV_CONCURRENT_BUILDS=2
```

A lower value reduces peak memory consumption, although dependency installation may take longer.

**Important:** `UV_CONCURRENT_BUILDS` controls concurrent source builds performed by uv. It does not control the number of C++ compiler processes started by CMake or Make.

### 15.2 Control C++ compilation parallelism

The `MAX_JOBS` environment variable controls the number of parallel jobs used by vLLM's build configuration.

For a machine with limited memory:

```bash
export MAX_JOBS=2
```

For a more capable machine:

```bash
export MAX_JOBS=4
```

Avoid automatically setting this to the total number of logical CPU cores on machines with limited RAM.

### 15.3 Control CMake parallelism

Set:

```bash
export CMAKE_BUILD_PARALLEL_LEVEL=2
```

This limits the parallelism used by CMake when it invokes supported build tools.

### 15.4 Control Rust compilation parallelism

Rust compilation can also consume significant memory.

Set:

```bash
export CARGO_BUILD_JOBS=2
```

For machines with very limited RAM:

```bash
export CARGO_BUILD_JOBS=1
```

### 15.5 Recommended settings

| Available RAM | UV_CONCURRENT_BUILDS | MAX_JOBS | CMAKE_BUILD_PARALLEL_LEVEL | CARGO_BUILD_JOBS |
|---|---:|---:|---:|---:|
| 8 GB | 1 | 1 | 1 | 1 |
| 16 GB | 1 | 2 | 2 | 2 |
| 32 GB | 2 | 4 | 4 | 2 |
| 64 GB or more | 4 | 8 | 8 | 4 |

These are conservative starting points, not guaranteed memory limits. Actual memory consumption varies with architecture, dependency versions, and build targets.

For an 8 GB machine, adding swap space can also help prevent OOM termination. Swap is slower than RAM and will increase build time.

### 15.6 Apply limited-memory settings

For a low-memory system, use the following configuration before installing dependencies and compiling:

```bash
export UV_CONCURRENT_BUILDS=1
export MAX_JOBS=1
export CMAKE_BUILD_PARALLEL_LEVEL=1
export CARGO_BUILD_JOBS=1
```

Verify:

```bash
echo "UV_CONCURRENT_BUILDS=$UV_CONCURRENT_BUILDS"
echo "MAX_JOBS=$MAX_JOBS"
echo "CMAKE_BUILD_PARALLEL_LEVEL=$CMAKE_BUILD_PARALLEL_LEVEL"
echo "CARGO_BUILD_JOBS=$CARGO_BUILD_JOBS"
```

These settings should be applied before running the uv dependency installation and build commands.

---

## 16. Install the generated vLLM wheel

After a successful build, install the generated wheel into the active Python virtual environment.

```bash
cd "$HOME/src/vllm"

uv pip install "$HOME"/src/vllm/dist/*.whl
```

Verify the installation:

```bash
python -c "import vllm; print('vLLM version:', vllm.__version__)"
```

Verify the CPU PyTorch configuration:

```bash
python -c "import torch; print('PyTorch:', torch.__version__); print('CUDA available:', torch.cuda.is_available())"
```

Check the vLLM executable:

```bash
which vllm
vllm --help
```

The installed executable should resolve to the virtual environment:

```text
$HOME/.venvs/vllm/bin/vllm
```

---

## 17. Run vLLM locally

Once installation is successful, vLLM can be launched using the CPU backend.

For example:

```bash
vllm serve <model-name> \
    --device cpu
```

Replace `<model-name>` with a supported Hugging Face model identifier or a local model path.

For a local model:

```bash
vllm serve /path/to/local/model \
    --device cpu
```

The server can be tested using:

```bash
curl http://localhost:8000/health
```

The OpenAI-compatible API endpoint is:

```text
http://localhost:8000/v1
```

**Note:** Model compatibility, memory requirements, CPU instruction support, and supported quantization formats depend on the selected model and the vLLM revision.

---

## 18. Useful environment configuration

The following environment variables can be added to `~/.bashrc` to make the configuration persistent.

```bash
export VLLM_TARGET_DEVICE=cpu

export CC=/usr/bin/gcc-15
export CXX=/usr/bin/g++-15

export BINUTILS_PREFIX="$HOME/.local/opt/binutils-2.47"

export CCACHE_DIR="$HOME/.cache/ccache"

export PATH="$HOME/.local/bin:$HOME/.cargo/bin:$BINUTILS_PREFIX/bin:$PATH"

export UV_HTTP_TIMEOUT=500
export UV_INDEX_STRATEGY=unsafe-best-match
export UV_LINK_MODE=copy

export CMAKE_CXX_COMPILER_LAUNCHER=ccache
```

Add the configuration:

```bash
cat >> "$HOME/.bashrc" <<'EOF'

# vLLM CPU build configuration
export VLLM_TARGET_DEVICE=cpu
export CC=/usr/bin/gcc-15
export CXX=/usr/bin/g++-15
export BINUTILS_PREFIX="$HOME/.local/opt/binutils-2.47"
export CCACHE_DIR="$HOME/.cache/ccache"
export PATH="$HOME/.local/bin:$HOME/.cargo/bin:$BINUTILS_PREFIX/bin:$PATH"
export UV_HTTP_TIMEOUT=500
export UV_INDEX_STRATEGY=unsafe-best-match
export UV_LINK_MODE=copy
export CMAKE_CXX_COMPILER_LAUNCHER=ccache
EOF
```

Reload:

```bash
source "$HOME/.bashrc"
```

Memory-specific settings such as `MAX_JOBS` and `UV_CONCURRENT_BUILDS` can be configured separately depending on the machine.

---

## 19. Troubleshooting

### 19.1 Python executable not found

Activate the virtual environment:

```bash
source "$HOME/.venvs/vllm/bin/activate"
```

Check:

```bash
which python
python --version
```

### 19.2 GCC 15 not found

Check whether GCC 15 is installed:

```bash
dpkg -l | grep gcc-15
```

Install:

```bash
sudo apt-get update
sudo apt-get install -y gcc-15 g++-15
```

### 19.3 uv command not found

Update PATH:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### 19.4 Rust or Cargo command not found

Load the Rust environment:

```bash
source "$HOME/.cargo/env"
```

Check:

```bash
rustc --version
cargo --version
```

### 19.5 Build process terminated unexpectedly

Check system memory:

```bash
free -h
```

Check for OOM events:

```bash
sudo dmesg -T | grep -i -E 'out of memory|oom|killed process'
```

Reduce build parallelism:

```bash
export UV_CONCURRENT_BUILDS=1
export MAX_JOBS=1
export CMAKE_BUILD_PARALLEL_LEVEL=1
export CARGO_BUILD_JOBS=1
```

Retry the failed build stage.

### 19.6 Binutils version mismatch

Check:

```bash
which as
as --version
```

Ensure the custom Binutils directory appears before `/usr/bin` in PATH:

```bash
export PATH="$BINUTILS_PREFIX/bin:$PATH"
```

### 19.7 CPU-only PyTorch verification

Run:

```bash
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"
```

Expected:

```text
False
```

### 19.8 Check compiler cache

```bash
ccache -s
```

Clear the cache if necessary:

```bash
ccache -C
```

---

## 20. Complete installation verification

Run the following script to verify the environment.

```bash
#!/usr/bin/env bash

set -e

echo "======================================"
echo " vLLM CPU BUILD ENVIRONMENT"
echo "======================================"

echo
echo "===== OPERATING SYSTEM ====="
cat /etc/os-release | grep PRETTY_NAME

echo
echo "===== CPU ARCHITECTURE ====="
uname -m

echo
echo "===== CPU INFORMATION ====="
nproc
lscpu | head -n 15

echo
echo "===== MEMORY ====="
free -h

echo
echo "===== COMPILER ====="
gcc-15 --version
g++-15 --version

echo
echo "===== BINUTILS ====="
as --version | head -n 1

echo
echo "===== PYTHON ====="
python --version
which python

echo
echo "===== UV ====="
uv --version

echo
echo "===== RUST ====="
rustc --version
cargo --version

echo
echo "===== BUILD TOOLS ====="
cmake --version
ninja --version
make --version

echo
echo "===== SYSTEM UTILITIES ====="
jq --version
git --version
ccache --version | head -n 1

echo
echo "===== PYTORCH ====="
python -c "import torch; print('Version:', torch.__version__); print('CUDA available:', torch.cuda.is_available())"

echo
echo "===== VLLM ====="
python -c "import vllm; print('Version:', vllm.__version__)"

echo
echo "======================================"
echo " ENVIRONMENT VERIFICATION COMPLETED"
echo "======================================"
```

---

## 21. CPU-only build considerations

1. No CUDA Toolkit, CUDA runtime, NVIDIA drivers, or NCCL installation is required.
2. The CPU build uses the CPU-specific dependency files maintained in the repository.
3. GCC 15 and G++ 15 are used to match the Dockerfile build configuration.
4. GNU Binutils 2.47 provides the newer assembler required by the AMX-FP8 build configuration.
5. Rust frontend artifacts are built using the repository's Rust configuration.
6. `ccache` can reduce subsequent compilation time when rebuilding after source changes.
7. `UV_CONCURRENT_BUILDS` should be reduced on machines with limited RAM.
8. `MAX_JOBS`, `CMAKE_BUILD_PARALLEL_LEVEL`, and `CARGO_BUILD_JOBS` should also be configured according to available memory.
9. CPU ISA optimizations must match the target processor. Do not enable x86-specific ISA flags on ARM64.
10. The generated wheel is specific to the Python ABI, architecture, and build configuration. Rebuild it when moving to an incompatible target environment.

---

## 22. Reference directory structure after installation

The expected directory structure is:

```text
$HOME/
|
+-- src/
|   |
|   +-- vllm/
|       |
|       +-- requirements/
|       +-- tools/
|       +-- rust/
|       +-- vllm/
|       +-- setup.py
|       +-- rust-toolchain.toml
|       +-- docker/
|       +-- dist/
|           |
|           +-- vllm-*.whl
|
+-- .venvs/
|   |
|   +-- vllm/
|
+-- .local/
|   |
|   +-- bin/
|   |   +-- uv
|   |
|   +-- opt/
|       |
|       +-- binutils-2.47/
|
+-- .cargo/
|   |
|   +-- bin/
|
+-- .cache/
    |
    +-- uv/
    +-- ccache/
```

---

## 23. Final build summary

The complete native CPU build process consists of the following stages:

| Stage | Description |
|---|---|
| 1 | Install Ubuntu system dependencies |
| 2 | Install GCC 15 and G++ 15 |
| 3 | Build GNU Binutils 2.47 |
| 4 | Install uv |
| 5 | Install Python 3.12 |
| 6 | Create the Python virtual environment |
| 7 | Install Rust build dependencies |
| 8 | Clone the vLLM repository |
| 9 | Configure the CPU build environment |
| 10 | Install CPU-only PyTorch dependencies |
| 11 | Install CPU build dependencies |
| 12 | Build the Rust frontend |
| 13 | Build the vLLM wheel |
| 14 | Install the generated wheel |
| 15 | Verify the local installation |

**Result:** A locally installed, CPU-only vLLM environment built from source on Ubuntu, without requiring Docker or any NVIDIA/CUDA components.

For reproducible builds, always record the Git commit, Ubuntu release, CPU architecture, Python version, GCC version, and build configuration used to generate the wheel.