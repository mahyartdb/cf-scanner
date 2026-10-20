# Contributing to cf-scanner

Thanks for helping. This is a solo-maintained Rust CLI — small, focused, and
strict about its contracts. Read this before opening a PR.

## Setup

The toolchain is pinned to **Rust 1.88** (edition 2024) by
`rust-toolchain.toml`. Always pass `--locked`; the lockfile is tracked on
purpose.

```bash
git clone https://github.com/QMahyar/cf-scanner.git
cd cf-scanner
cargo build --all-targets --locked
```

The build downloads the pinned DB-IP GeoIP database into `OUT_DIR`. Without
network access, use `CFSCANNER_OFFLINE_BUILD=1` to embed a placeholder
(country lookups then return `None`).

## The gate

CI runs the same commands on ubuntu, windows and macos. A PR is green when
all of these pass:

```bash
cargo build --all-targets --locked
cargo test --locked
cargo test --doc --locked
cargo clippy --all-targets --locked -- -D warnings
cargo check --all-features --locked
cargo fmt --check
```

Plus two repo-level guards in CI:

- **placeholder-guard** — no tracked file may exceed 1 MB. Never commit the
  real xray binary or a `.mmdb`; the tracked `data/bundled/xray*` files are
  0-byte placeholders. If a `dist-bundle-xray` build overwrote them, restore
  them with `git restore data/bundled/xray data/bundled/xray.exe`.
- **version-parity** — `Cargo.toml` version == `npm/cf-scanner/package.json`
  version == `RELEASE_TAG` in `npm/cf-scanner/install.js`. All three must
  change together.

Coverage is gated at 80% lines (`cargo llvm-cov ... --fail-under-lines 80`).

## Contracts that must not regress

These are the load-bearing behaviours. Changing any of them is a breaking
change and needs a note in the PR body.

1. **stdout is NDJSON only** (`scan`, `check-sub` events plus a final summary
   object). **stderr is human progress.** `--verbose` adds per-IP diagnostics
   on stderr only. `--json-errors` adds `{"type":"error",...}` on stdout.
2. **`ScanConfig::validate()` is the single admission gate** — both the CLI and
   the retry-load path go through it.
3. **Phase2/WARP sub-configs are `deny_unknown_fields`.** A new field needs
   `#[serde(default)]`. The top-level `ScanConfig` is intentionally *not*
   strict, for `--retry-last` forward compatibility.
4. **Exports are atomic** (tmp + rename, `0600` / owner-only DACL on Windows).
   `--export-live` is a crash-safe append with fsync. The two conflict, and
   `--export-live -` is rejected.
5. **Secrets are owner-only at creation** (`paths::write_secret`: `0600`/`0700`
   on Unix, protected DACL via `CreateFile2` on Windows). Configs, keys and raw
   URLs are never logged; errors are sanitized.
6. **Bundle export formats are IPv4-only.** IPv6 rows warn and skip.
   `write_export` / `render_bundle` must never silently write an empty bundle
   when passing endpoints existed.
7. **The export registry (`src/export.rs::FORMATS`) is the single source of
   truth** for format names and help text. A new `FORMATS` row without a
   renderer fails a test on purpose — implement the renderer too.

## Tests

- New logic in layers that already have a harness gets a **failing test
  first**, then the implementation. UI, styles and scripts are test-after.
- Offline E2E uses `203.0.113.0/30` plus `--seed` for determinism
  (`tests/cli_scan_agent.rs`). The CSV header and tmp-cleanup are pinned by
  tests.
- `#[ignore]` live tests need a real `CFSCANNER_SUB_URL` (a credential —
  never log it):
  ```bash
  cargo test --locked --test live_smoke -- --ignored --nocapture
  ```
  CI runs them only on dispatch/schedule, with the secret.

## Commits and pull requests

- One logical change per commit. Conventional prefix
  (`feat|fix|refactor|test|docs|chore:`) plus what-and-why in one line.
- Land on `main` unless the work is big or risky enough to need a branch.
- Never stage unrelated files. Never commit a secret, a real binary or a
  `.mmdb`.
- Version bumps and tags are the maintainer's call — open a PR, don't tag.

## Release

Releases are driven by `cargo-dist` 0.32.0 (`dist-workspace.toml`). Pushing a
tag matching `*[0-9].[0-9].[0-9]*` runs plan → gate → cross-check → build →
host → npm-publish. Never publish manually and never bump a version without
explicit approval for that version.
