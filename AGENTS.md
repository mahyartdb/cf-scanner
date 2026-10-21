# AGENTS.md

Pure Rust CLI (`cf-scanner`): finds working Cloudflare CDN / WARP endpoints. No server, UI, or tray. Entry `src/main.rs` (thin runner); clap types `src/cli.rs` + `src/cli/scan_args.rs`; public facade `src/lib.rs` (`geo`, `socks`, `inline_verify` are crate-private).

## Commands (use `--locked` everywhere)

```
cargo build --all-targets --locked
cargo test --locked                    # unit + integration
cargo test --doc --locked
cargo test <name> -- --nocapture       # single unit test
cargo test --test cli_scan_agent <name> # single E2E test
cargo clippy --all-targets --locked -- -D warnings
cargo check --all-features --locked
cargo fmt --check
cargo llvm-cov --all-targets --locked --lcov --output-path lcov.info --fail-under-lines 80
```

Toolchain pinned to 1.88 by `rust-toolchain.toml` (edition 2024). CI runs the same gate on ubuntu + windows + macos.

## Build-time network (build.rs)

- Downloads DB-IP mmdb pinned by `data/geoip-version.txt` (version line + sha line) via `curl` (must be in PATH); verifies sha, 64 MiB caps, caches in `OUT_DIR`. Fail-closed on mismatch.
- `CFSCANNER_OFFLINE_BUILD=1` embeds a placeholder instead (country lookups return `None`).
- `--features dist-bundle-xray` downloads the Xray zip pinned by `data/xray-version.txt`, verifies `.dgst` (`SHA2-256= <64hex>` grammar in `src/dgst.rs`), extracts into `data/bundled/`. Normal builds do not bundle.
- `data/bundled/xray` and `data/bundled/xray.exe` are 0-byte placeholders. Never commit real binaries/mmdb: CI fails any tracked file >1 MB. Restore with `git restore data/bundled/xray data/bundled/xray.exe`.

## Pins that must stay in sync

- Version triple: `Cargo.toml` version == `npm/cf-scanner/package.json` version == `RELEASE_TAG` in `npm/cf-scanner/install.js`. CI `version-parity` fails otherwise. Current `0.15.2`.
- Xray pin: `data/xray-version.txt` (vX.Y.Z) must be read by both `src/xray.rs` and `build.rs`; CI `xray-parity` greps both and checks the `XTLS/Xray-core` release tag exists.

## Runtime I/O contract (do not regress)

- stdout = NDJSON events only (`scan`, `check-sub`), ending with a summary object. stderr = human progress/diagnostics.
- `--verbose` adds per-IP diagnostic lines on stderr only. `--json-errors` adds `{"type":"error",...}` on stdout on failure (stderr keeps human text).
- `--export FILE --export-format <name>` is atomic tmp+rename (`0600`, owner-only DACL on Windows). `--export-live FILE` is a crash-safe streaming write with per-row fsync (truncates an existing file). The two conflict; `--export-live -` is rejected (`-` = stdout). Formats registry is `src/export.rs::FORMATS`; help text derives from it.

## Architecture notes

- One engine: `src/engine/` (`driver`, `cdn`, `warp`, `phase2`, `plan`, `store`, `neighbor`, `speed`) behind `ScanController`. Probe transports are trait-injectable so tests stay offline.
- `ScanConfig::validate()` is the single admission gate (CLI + retry-load). `Phase2/Warp` sub-configs are `deny_unknown_fields` (new fields need `#[serde(default)]`); top-level `ScanConfig` is not (forward-compat for `--retry-last` persistence).
- `src/export.rs` `write_export`/`render_bundle` never writes an empty bundle silently when passing endpoints existed; bundle formats are IPv4-only (IPv6 passing rows warn + skip).
- `CF_SCANNER_DATA_DIR` overrides the data dir (ranges cache, xray binary, retry config). Tests must use `paths::test_env::IsolatedDataDir` + `DATA_DIR_LOCK`, lock order `IDENTITY_LOCK` before `DATA_DIR_LOCK`.
- Secrets (`identity`, trial xray configs, exports) are owner-only at creation: `paths::write_secret` (`0600`/`0700` Unix, protected DACL via `CreateFile2` on Windows). Never log configs, keys, or raw URLs (errors are sanitized).

## Tests

- Offline E2E uses `203.0.113.0/30` + `--seed` for determinism (`tests/cli_scan_agent.rs`); CSV header and tmp-cleanup are pinned by tests.
- `#[ignore]` live tests need `CFSCANNER_SUB_URL` (a credential, never log it): `cargo test --locked --test live_smoke -- --ignored --nocapture` (same for `cli_scan_agent`). CI `live-evidence.yml` runs them only on dispatch/schedule with the secret.

## Release

- `cargo-dist 0.32.0` via `dist-workspace.toml`: targets `aarch64-linux`, `x86_64-linux`, `x86_64-windows-msvc`; installers `msi`, `powershell`, `shell`. Tag push matching `*[0-9].[0-9].[0-9]*` drives plan → gate → cross-check → build → host → npm-publish. Never publish manually; never bump versions/tags without explicit user approval for that version.
