# smolvm — Codebase Map

Depth cap 2. Top-level directories of the Rust workspace.

| Path | What it is |
|---|---|
| `src/` | Root crate `smolvm` — CLI entry (`main.rs`, `cli/`), host runtime: machine lifecycle (`pool.rs`, `db.rs`, `image_store.rs`), agent launcher (`agent/`), internal boot (`internal_boot.rs`), platform shims (`platform/`), networking (`network/`), API surface (`api/`), CUDA host side (`cuda_host.rs`), secrets, settings, checkpoint/pack export |
| `crates/` | 15 workspace member crates: `smolvm-agent` (guest daemon), `smolvm-cuda*` / `smolvm-cudart-shim` / `smolvm-nvml-shim` (GPU shims), `smolvm-network`, `smolvm-oci-layer`, `smolvm-pack` (.smolmachine), `smolvm-protocol`, `smolvm-registry`, `smolvm-s3fs`, `smolvm-shim`, `smolvm-smolfile` (Smolfile parsing) |
| `lib/` | Prebuilt bundled hypervisor libs (git-lfs): `libkrun`/`libkrunfw` for macOS in `lib/`, Linux `.so` in `lib/linux-x86_64/` and `lib/linux-aarch64/`, Windows DLLs in `lib/windows-x86_64/` |
| `libkrun/`, `libkrunfw/` | Vendored sources/build glue for libkrun and the guest kernel blob (submodules + nix builds; rarely touched locally) |
| `scripts/` | Build/install helpers: `build-agent-rootfs.sh`, `build-dist.sh`, `build-libkrun-linux.sh`, `cut-release.sh`, `install-local.sh`, guard scripts checked in CI |
| `tests/` | Integration suites run by `tests/run_tests.sh` (cli, pack, machine, api, network, storage, …), harness in `tests/common.sh`; Python API tests need a running `smolvm serve` |
| `docs/` | `DEVELOPMENT.md` (start here), platform install guides, incremental checkpoints, hardware video |
| `sdks/` | Client SDKs (`sdks/node`, `sdks/python`) for the `smolvm serve` API |
| `smolvm-sdk/` | SDK-related sources |
| `bench/` | Startup/fork/pool-memory benchmarks (`bench.sh`, result JSONs) |
| `deploy/` | Kubernetes/k3s manifests for the registry/server deployment |
| `nix/` | Nix expressions for libkrun, libkrunfw, smolvm |
| `packaging/`, `demo/`, `examples/`, `assets/` | Distro packaging, demos, examples, logo |
| `Makefile.toml` | cargo-make task definitions (dev, smolvm, test*, lint, dist, agent*) |
| `.github/workflows/` | CI: fmt, secrets guards, libkrun provenance, audit, cross-platform builds, release |
