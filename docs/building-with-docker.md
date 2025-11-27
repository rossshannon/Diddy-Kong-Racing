# Building with Docker

This document describes how to build the Diddy Kong Racing ROM entirely inside a Docker container, without installing any MIPS cross toolchains on your host system.

The approach is:

- Use a Linux container (Ubuntu 24.04, amd64) as the build environment.
- Avoid distro `binutils-mips-linux-gnu`; instead, use the repo’s own `tools/get-binutils.sh` to build a local `mips64-elf` binutils.
- Work around a Python tooling incompatibility by constraining the `rabbitizer` version used by `spimdisasm`/`splat64`.
- Mount your working tree into the container so the final ROM appears in `build/dkr.us.v77.z64` on your host.

## Prerequisites

- Docker Desktop (or Docker Engine) installed.
- Your `baseroms/` directory already contains a valid DKR ROM, as required by the main README.
- You are running these commands from the repo root, e.g.

```bash
cd /path/to/Diddy-Kong-Racing
```

All commands below assume your current directory is the repo root.

## Dockerfile (nocross)

This project includes a Dockerfile that intentionally **does not** install any system MIPS cross toolchains. Instead, the Makefile will fall back to `tools/get-binutils.sh`, which builds `mips64-elf` binutils into `tools/binutils/`.

The file is `Dockerfile.nocross` in the repo root. It:

- Uses `ubuntu:24.04` as the base image.
- Installs build dependencies:
  - `build-essential`, `make`, `pkg-config`, `git`
  - `python3`, `python3-pip`, `python3-venv`
  - `libpcre2-dev`, `libpcre2-8-0`, `zlib1g-dev`
  - `curl`, `wget`, `xz-utils`, `ca-certificates`
- **Does not** install `binutils-mips-linux-gnu` or any `gcc-mips-*` packages.

This ensures the Makefile’s tool-detection logic chooses the repo’s own MIPS toolchain rather than a potentially incompatible system one.

## Build the Docker image

Build the image as amd64 (x86_64) so it matches common Linux setups for this repo:

```bash
docker build \
  --platform linux/amd64 \
  -f Dockerfile.nocross \
  -t dkr-builder-nocross .
```

This may take a minute the first time while Ubuntu and build dependencies are downloaded.

## One-shot build command

Run the full build inside the container with a single `docker run` invocation:

```bash
docker run --rm \
  --platform linux/amd64 \
  -u "$(id -u):$(id -g)" \
  -v "$PWD":/work \
  -w /work \
  dkr-builder-nocross \
  bash -lc '
    # Start from a clean Python virtualenv
    rm -rf .venv

    # Constrain rabbitizer to a version compatible with spimdisasm
    cat > /tmp/pip-constraints.txt <<EOF
rabbitizer==1.13.0
EOF
    export PIP_CONSTRAINT=/tmp/pip-constraints.txt

    # 1) Rebuild host tools (n64crc, dkr_assets_tool)
    make -C tools clean && \

    # 2) Project setup: create .venv, install Python deps, fetch IDO
    make setup && \

    # 3) Build local mips64-elf binutils into tools/binutils/
    bash -eo pipefail tools/get-binutils.sh && \

    # 4) Extract assets from the baserom
    make extract && \

    # 5) Build the game ROM
    make -j"$(nproc)"
  '
```

What this does, in order:

1. **Cleans `.venv`** to ensure a fresh Python environment.
2. Creates `/tmp/pip-constraints.txt` with:

   ```text
   rabbitizer==1.13.0
   ```

   and sets `PIP_CONSTRAINT` so that `make setup` installs this rabbitizer version, avoiding a runtime `AttributeError` in `spimdisasm`.

3. Runs `make -C tools clean` to force `n64crc` and `tools/dkr_assets_tool` to be rebuilt for Linux/amd64 inside the container.
4. Runs `make setup`:
   - Creates `.venv/` using `python3 -m venv`.
   - Installs `splat64==0.35.2`, `spimdisasm==1.36.1`, and other Python deps under the rabbitizer constraint.
   - Downloads and unpacks the IDO static recompiler into `tools/ido-recomp/linux`.
5. Runs `tools/get-binutils.sh`:
   - Downloads and builds GNU binutils 2.36 for target `mips64-elf`.
   - Installs `mips64-elf-{ar,as,ld,objcopy,objdump,strip}` into `tools/binutils/`.
   - Because the container has **no** `binutils-mips-linux-gnu` package, the Makefile will choose these tools (`CROSS := tools/binutils/mips64-elf-`).
6. Runs `make extract`:
   - Uses `splat64` to split the baserom based on the `ver/splat/dkr.us.v77.yaml` config.
   - Uses `tools/dkr_assets_tool extract -dkrv us.v77` to extract and convert assets.
   - Produces `assets/assets.bin` and updates `include/asset_enums.h`.
7. Runs `make -j$(nproc)` to build the game:
   - Compiles all C and ASM sources.
   - Links the ELF and ROM images via the `mips64-elf-ld` built in step 5.

Because your repo is bind-mounted into `/work`, all outputs appear on your host filesystem.

## Verifying the result

On a successful run, you should see the `verify` step print something like:

```text
CRC 1: 0x53D440E7 (Good)
CRC 2: 0x7519B011 (Good)
Verify: OK
```

The final ROM will be at:

- `build/dkr.us.v77.z64`

from the repo root on your host.

You can now point your N64 emulator at that file.

## Notes and caveats

- **Performance:** On Apple Silicon, running an amd64 Ubuntu image goes through emulation, so builds will be slower than on native Linux.
- **pip cache warnings:** You may see warnings about `/.cache/pip` not being writable. These are harmless in this setup; pip simply disables its cache inside the container.
- **Toolchain warnings:** Building binutils will emit various warnings (missing `makeinfo`, deprecated APIs, etc.). As long as the script completes without `Error` lines, those warnings are expected.
- **Host files:** This process does not modify any project files beyond normal build artifacts (`build/`, `assets/`, `.venv/`, `tools/binutils/`, etc.). No Makefiles or sources are changed.
