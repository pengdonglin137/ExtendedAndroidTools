# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ExtendedAndroidTools is a GNU Make-based cross-compilation build system for Linux tools (bpftrace, bcc, llvm, python, etc.) targeting Android. Each tool is built using its native build system (autotools, cmake) via the Android NDK. The reference build environment is Docker (Ubuntu Jammy).

## Build Commands

### Docker (recommended)
```bash
./scripts/build-docker-image.sh          # Build the Docker image
./scripts/run-docker-build-env.sh         # Enter the build container
make bpftrace                             # Build bpftrace for Android (default arm64)
make bpftools                             # Build full bpftools sysroot (bpftrace+bcc+python+xz)
make bpftools-min                         # Build minimal sysroot (bpftrace only, no python/bcc)
make python-host                          # Build python for host platform
eval `make setup-env`                     # Set PATH for host-built tools
```

### Build variables
```bash
make bpftools NDK_ARCH=x86_64 BUILD_TYPE=Debug THREADS=8
```
- `NDK_ARCH`: arm64 (default), x86_64, armv7
- `BUILD_TYPE`: Release (default), Debug — controls debug info
- `THREADS`: parallel jobs, default 4
- `NDK_PATH`: NDK location, default `/opt/ndk/android-ndk-r27b` (set in `toolchains/toolchains.mk`)
- `STATIC_LINKING`: true/false, static link non-NDK deps
- `LLVM_BPF_ONLY`: true/false, only build BPF target for LLVM
- `BPFTRACE_NO_STRIP`: true/false, keep debug symbols in bpftrace release builds

### Host tools
```bash
make python-host                          # Build for host platform
make cmake-host                           # Build cmake for host
```

### Cleanup
```bash
make clean                                # Remove build/ and out/
```

## Architecture

### Build system structure
```
Makefile              → top-level orchestrator
toolchains/toolchains.mk → NDK paths, arch-specific config (NDK_API=30)
toolchains/cmake.mk   → CMAKE variable + Android cross-compilation flags
toolchains/autotools.mk → autotools cross-compilation config
projects/project.mk   → project-define macro (sets up build targets/deps)
projects/*/build.mk   → per-project build rules
sysroot/bpftools.mk   → sysroot archive packaging
```

### Project convention
Each project in `projects/<name>/build.mk` follows this pattern:
1. Declares `<PROJECT>_ANDROID_DEPS` and `<PROJECT>_HOST_DEPS` (other projects it depends on)
2. Calls `$(eval $(call project-define,<name>))` which auto-generates:
   - `<name>` → builds for Android (phony target)
   - `<name>-host` → builds for host
   - `prepare-<name>` → generates build directory (configure/cmake)
3. Defines build directory rule (cmake or autotools configure)
4. Defines install rule (copy to `out/android/$ARCH/`)

### Output layout
```
out/android/<arch>/bin/    → binaries
out/android/<arch>/lib/    → shared libraries
out/android/<arch>/include/ → headers
out/sysroots/<arch>/       → packaged sysroot archives
out/host/                  → host-built tools (flex, cmake, python)
build/android/<arch>/      → build directories per project
```

### Sysroot packaging
`make bpftools` produces `bpftools-<arch>.tar.gz` containing:
- Binaries: bpftrace, bpftrace-aotrt, python3, xzcat
- Libraries: libbcc, libbpf, libclang, libelf, libfl, liblzma, libffi, libc++_shared
- Wrapper scripts for runtime library path setup
- `setup.sh` for configuring environment on device

### Dependency chain
Projects declare dependencies via `_ANDROID_DEPS` and `_HOST_DEPS`. For example:
- `bpftrace` depends on: bcc, cereal, elfutils, flex, libbpf, llvm, stdc++fs, xz
- `bcc` depends on: llvm, libbpf, flex, elfutils, python, xz

The `project-define` macro in `projects/project.mk` resolves these into proper Make dependency rules.

## Device deployment
```bash
adb push bpftools-arm64.tar.gz /data/local/tmp
adb shell "cd /data/local/tmp && tar xf bpftools-arm64.tar.gz"
adb shell /data/local/tmp/bpftools/bpftrace -e 'uprobe:/system/lib64/libc.so:malloc { @ = hist(arg0); }'
```
Requires root and BPF-enabled kernel (BPF + Uprobes + Kprobes).
