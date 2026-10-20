## What and why

<!-- One paragraph: what changes and why it is needed. -->

## Checklist

- [ ] `cargo build --all-targets --locked`
- [ ] `cargo test --locked`
- [ ] `cargo test --doc --locked`
- [ ] `cargo clippy --all-targets --locked -- -D warnings`
- [ ] `cargo check --all-features --locked`
- [ ] `cargo fmt --check`
- [ ] New logic has a test (failing test first, for layers with a harness)
- [ ] No secret, real binary or `.mmdb` is committed
- [ ] `data/bundled/xray` and `data/bundled/xray.exe` are still 0-byte placeholders
      (`git restore data/bundled/xray data/bundled/xray.exe` if a bundle build overwrote them)
- [ ] Version triple updated together, if this changes the version:
      `Cargo.toml` == `npm/cf-scanner/package.json` == `RELEASE_TAG` in `npm/cf-scanner/install.js`

## Contracts touched

<!--
Tick only what this PR changes, and say how it was verified. Leave a note if
one of these is deliberately untouched.
-->

- [ ] stdout/stderr split (NDJSON on stdout, human progress on stderr)
- [ ] `ScanConfig::validate()` admission gate
- [ ] Export atomicity (`--export` / `--export-live`)
- [ ] Secret file permissions (`paths::write_secret`)
- [ ] Export format registry (`src/export.rs::FORMATS`)

## Breaking changes

<!-- Does this change the CLI surface, the output contract or the config format? -->
