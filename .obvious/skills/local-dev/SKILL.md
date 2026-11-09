---
name: local-dev
description: How to get smolvm local dev running from a clean sandbox (proven during onboarding, 2026-11-09)
---

# local-dev: smolvm

## What the app is

Rust workspace CLI (`smolvm`) that runs Linux microVMs via libkrun/KVM. No
daemons, no external databases. "Healthy local dev" = workspace builds, unit
tests pass, and an ephemeral VM boots and executes a command.

## Steps that worked (clean sandbox)

1. **git-lfs first.** `lib/` and `lib/linux-x86_64/` are git-lfs pointers until
   pulled. Sandbox has no sudo/apt, so install userland git-lfs:
   ```bash
   curl -sL https://github.com/git-lfs/git-lfs/releases/download/v3.7.0/git-lfs-linux-amd64-v3.7.0.tar.gz | tar xz -C ~/tools
   export PATH=~/tools/git-lfs-3.7.0:$PATH
   git lfs install --skip-repo && git lfs pull
   ```
   Verify `lib/linux-x86_64/libkrun.so` is a real ELF (multi-MB), not 132 bytes.

2. **Rust toolchain** (userland):
   ```bash
   curl -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable --profile minimal
   . "$HOME/.cargo/env"
   rustup target add x86_64-unknown-linux-musl   # for the agent rootfs build
   ```
   cargo-make is optional; plain cargo works (`cargo make dev` is only needed
   for the signed release build).

3. **Build** (~3 min on 2 cores): `cargo build --workspace` → `target/debug/smolvm`.

4. **musl toolchain for the agent rootfs.** The sandbox has no
   `musl-gcc` and no sudo/docker, so compile musl into the home dir (~13s):
   ```bash
   curl -sL https://musl.libc.org/releases/musl-1.2.5.tar.gz | tar xz -C /tmp
   cd /tmp/musl-1.2.5 && ./configure --prefix=$HOME/musl && make -j2 && make install
   export CC_x86_64_unknown_linux_musl=$HOME/musl/bin/musl-gcc
   export AR_x86_64_unknown_linux_musl=/usr/bin/ar   # musl ships no musl-ar; GNU ar works
   ```

5. **Agent rootfs** (~2 min): `./scripts/build-agent-rootfs.sh` →
   `target/agent-rootfs/` (~34M). Harmless failure at the end: "Vulkan guest
   driver fetch failed — rootfs ships without auto-staging" (needs bsdtar/rpm2cpio).

6. **Run VMs.** lib discovery resolves `SMOLVM_LIB_DIR` or exe-relative
   `../../lib/linux-<arch>` — from `target/debug/smolvm` it just works:
   ```bash
   export SMOLVM_AGENT_ROOTFS=$PWD/target/agent-rootfs
   ./target/debug/smolvm machine run --net --image alpine -- sh -c "uname -a"
   ```

## Proof captured at onboarding

- Ephemeral VM: printed `Hello world from a microVM` + guest kernel `6.12.95`.
- Persistent VM: `create --name obonboard --image alpine` → `start` → `exec`
  install python3 → install survived a second `exec` and a `stop`/`start` →
  machine deleted.
- `cargo test --lib`: 929 passed, 0 failed, 1 ignored.
- `cargo fmt --all -- --check`: clean. `cargo clippy --all-targets`: warnings
  only (pre-existing deprecated `fetch_update` calls), no errors.

## Requirements / environment

- `/dev/kvm` must exist (present in sandbox) for any machine/pack VM flow.
- Network needed for image pulls (registry) and first `apk` installs.
- No required secrets or env vars beyond `SMOLVM_AGENT_ROOTFS` (only because
  the binary runs from `target/debug/` with a local rootfs).

## Known traps

- LFS pointers instead of real libs → dlopen failures at boot.
- `machine delete` is interactive — pipe `echo y`.
- `tests/run_tests.sh` suites boot real VMs and take ~10 min total; use
  `cargo test --lib` for a fast gate.
- Don't use `DYLD_LIBRARY_PATH` on Linux (macOS-only cargo-make task var);
  Linux lib resolution is via `find_lib_dir()`/`SMOLVM_LIB_DIR`.
