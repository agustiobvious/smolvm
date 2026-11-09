# smolvm — Agent Guidance

## Repo

`agustiobvious/smolvm` — OCI-native microVM runtime. A Rust CLI (`smolvm`) that
builds and runs portable, self-contained Linux microVMs locally (<200ms boot via
libkrun), packs VMs into single-file `.smolmachine` artifacts, and ships SDKs.

- Default branch: `main`
- Language: Rust (workspace, edition 2021) — see `Cargo.toml` for the member crates
- Orchestration: [`cargo-make`](https://github.com/sagiegurari/cargo-make) (`Makefile.toml`)

## Stack

| Component | Notes |
|---|---|
| Rust stable | install via rustup; workspace has 16 crates |
| libkrun + libkrunfw | prebuilt, in `lib/` via **git-lfs** (macOS dylibs) and `lib/linux-x86_64/`, `lib/linux-aarch64/`, `lib/windows-x86_64/` |
| Hypervisor | KVM on Linux — requires `/dev/kvm` (present in this sandbox) |
| musl toolchain | needed to build the agent rootfs (see SKILL.md for the userland workaround) |
| git-lfs | required; bundled libs are LFS objects — run `git lfs pull` after clone |

No external services (no Postgres/Redis). "Local dev" = build the workspace,
build the agent rootfs, then run real microVMs against KVM.

## Commands

```bash
# Build (debug; ~3 min on 2 cores)
cargo build --workspace

# Build + codesign release binary (canonical dev build)
cargo make dev          # binary: ./target/release/smolvm

# Run the built binary with lib/agent-rootfs env wired up
cargo make smolvm <smolvm args>

# Unit tests (no VM required)
cargo make test-lib     # == cargo test --lib

# Full feature suites (~10 min, boots VMs)
cargo make test

# Lint
cargo make lint         # cargo fmt --check + clippy -D warnings

# Agent rootfs (needed once for machine/pack VM flows)
./scripts/build-agent-rootfs.sh   # output: target/agent-rootfs
```

Running a VM from a debug binary:

```bash
export SMOLVM_AGENT_ROOTFS=$PWD/target/agent-rootfs
./target/debug/smolvm machine run --net --image alpine -- sh -c "echo hi && uname -a"
```

## Codebase map

See `.obvious/codebase-map.md`.

## Local verification

1. `cargo build --workspace` — must finish without errors.
2. `cargo test --lib` — unit tests (929 passed at onboarding time).
3. `cargo make lint` (or `cargo fmt --all -- --check` + `cargo clippy --all-targets`).
4. Primary flow (requires `/dev/kvm` + agent rootfs): ephemeral VM run (above)
   must print the guest `uname` line; then a persistent
   create → start → exec → stop → start → exec cycle must keep installs.

## Snapshot

- snapshotId: `tensorlake-snapshot://artifact/e7rwnflyu29glpmwj58x9?source=rz1ra3twb1ml4nylz5yla&base=tensorlake%2Fubuntu-minimal&disk=40960`
- Captured: 2026-11-09T02:30:33.879Z
- State: repo @ main, toolchain + git-lfs + musl toolchain installed, workspace built, agent rootfs built, dev stack verified healthy.

## Gotchas

- Bundled `lib/` files are git-lfs pointers until `git lfs pull` — a stale pointer makes libkrun fail to load.
- No sudo/docker in the sandbox; build musl from source into `~/musl` and set
  `CC_x86_64_unknown_linux_musl`/`AR_x86_64_unknown_linux_musl` when building the agent rootfs.
- `machine delete` prompts; pipe `echo y` in scripts.
- Vulkan guest driver staging in the rootfs script fails without bsdtar/rpm2cpio — harmless for base flows.
- Pre-existing `cargo clippy` warnings exist (deprecated `fetch_update` calls in `src/pool.rs` region); don't treat as environment breakage.

## Validation Summary (onboarding, 2026-11-09)

- dev_stack_healthy: true
- `cargo build --workspace`: success (Rust 1.99.1, debug, ~3m01s)
- `cargo test --lib`: 929 passed / 0 failed / 1 ignored
- `cargo fmt --all -- --check`: clean
- `cargo clippy --all-targets`: finished; warnings only (pre-existing deprecations), no errors
- Primary flow (ephemeral VM): `machine run --net --image alpine` printed
  "Hello world from a microVM" + guest `uname` (kernel 6.12.95) via KVM
- Secondary flow (persistent VM): create → start → exec (python3 install)
  persisted across exec and stop/start; machine deleted
- snapshotId: tensorlake-snapshot://artifact/e7rwnflyu29glpmwj58x9 (2026-11-09T02:30:33.879Z)
