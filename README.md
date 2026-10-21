# cf-scanner

Find working Cloudflare CDN / WARP endpoints on restricted or throttled networks.

`cf-scanner` is a single static-ish Rust CLI. It sweeps the official Cloudflare IP
ranges, probes each candidate, optionally verifies the survivors through a real
proxy (xray), and exports the result in the format your client actually imports.

- **NDJSON on stdout, human progress on stderr** — pipe it straight into `jq`.
- **No server, no UI, no tray.** One binary, one command.
- **Offline-testable** — probe transports are trait-injectable, so the test suite
  never touches the network.

## Install

### npm (recommended)

```bash
npm install -g @qmahyar/cf-scanner
```

The `postinstall` script detects your OS/arch, downloads the matching prebuilt
archive from GitHub Releases, and unpacks the binary (plus its bundled xray
helper) into `bin/`. Set `CF_SCANNER_SKIP_INSTALL=1` to skip the download.

### Standalone binaries

Grab the archive for your platform from
[Releases](https://github.com/mahyartdb/cf-scanner/releases). Published targets:

| Platform | Target | Installers |
|---|---|---|
| Linux x64 | `x86_64-unknown-linux-gnu` | `shell`, `powershell` |
| Linux arm64 | `aarch64-unknown-linux-gnu` | `shell`, `powershell` |
| Windows x64 | `x86_64-pc-windows-msvc` | `msi`, `powershell`, `shell` |

macOS is not published (unsigned binaries); the runtime can still download xray
for macOS on demand.

### From source

```bash
cargo install --path . --locked
```

The build downloads the pinned DB-IP GeoIP database. On a machine without
network access, `CFSCANNER_OFFLINE_BUILD=1` embeds a placeholder instead —
country lookups then return `None` and everything else still works.

## Quick start

```bash
# 1 IP per /24 of the official ranges - fast sweep
cf-scanner scan --preset quick

# WARP endpoint discovery
cf-scanner scan --mode warp --count 512

# Verify candidates through a real proxy config
cf-scanner scan --phase2-configs vless://uuid@host:443 --phase2-fragment medium

# Re-run the last scan, re-supplying the phase-2 keys
cf-scanner scan --retry-last --phase2-configs vless://uuid@host:443

# Export results for another client
cf-scanner scan --preset quick --export results.csv
```

Not sure what to pick? `cf-scanner wizard` walks through the setup interactively.

## Commands

| Command | What it does |
|---|---|
| `scan` | Sweep Cloudflare ranges (CDN probe + optional xray phase 2) or WARP endpoints |
| `wizard` | Interactive guided scan setup |
| `ranges refresh` | Refresh the bundled Cloudflare IP range lists (`--ipv6` for the v6 pool) |
| `warp-config generate` | Generate a WARP WireGuard identity (wgconf) |
| `warp-config export` | Export that identity, optionally stamped with the best endpoint from the last scan |
| `check-sub <URL>` | Fetch a subscription URL and verify every config against its own server |
| `export-config` | Re-render a share URI so it dials a verified IP:port |
| `tune` | Find working junk / SNI / fragment settings by trial over small samples |

Run `cf-scanner <command> --help` for the full flag list.

### Useful scan flags

| Flag | Effect |
|---|---|
| `--mode cdn\|warp` | Scan Cloudflare ranges or discover WARP endpoints |
| `--preset quick\|normal\|full` | Sized sweep: 1 / 3 / every usable host per /24 |
| `--count N` | Probe N random candidates instead of a preset |
| `--target N` | Stop once N endpoints are found (default 20) |
| `--cap N` | Hard ceiling on probes sent |
| `--concurrency N` | Parallel probe workers (default 64, max 1000) |
| `--network-profile blocked\|slow` | Preset tuning for throttled/restricted networks; explicit flags always win |
| `--seed N` | Deterministic sampling — same seed, same plan |
| `--colo HKG,NRT` | Keep only phase-2 results in these Cloudflare colos |
| `--probe tcp\|tls\|http` | Phase-1 probe protocol (default `tls`) |
| `--speed-test` | Download an 8 MiB sample per verified endpoint and record MB/s |
| `--export FILE` | Write results to a file (atomic tmp+rename) |
| `--export-live FILE` | Stream NDJSON results as they arrive (crash-safe; truncates an existing file) |
| `--export-format <name>` | Output format, see below |

## Output contract

This is a stable contract — scripts depend on it.

- **stdout** carries NDJSON events only (`scan`, `check-sub`), ending with a
  summary object. `--export -` is rejected: the export blob would corrupt that
  stream, so exports always go to a file.
- **stderr** carries human progress and diagnostics.
- `--verbose` adds per-IP diagnostic lines **on stderr only**.
- `--json-errors` adds `{"type":"error",...}` on stdout when the program fails;
  stderr keeps the human-readable text.

### Export formats

`--export-format` accepts:

| Format | Shape |
|---|---|
| `csv` | Spreadsheet rows, one endpoint per line (default) |
| `json` | Full verdict objects with a count header |
| `raw` / `sharelinks` | Share URIs, one per line |
| `base64` | Base64 of the share-URI list |
| `singbox` | sing-box outbounds JSON |
| `clash` | Clash proxies JSON |
| `v2ray` | v2rayN clipboard JSON |
| `shadowrocket` | Base64 URI list for Shadowrocket import |
| `quantumult` | Quantumult X server lines |

Bundle formats are IPv4-only; IPv6 rows in a bundle are warned about and skipped.

## How it works

1. **Plan** — build the candidate list from the bundled Cloudflare ranges
   (`data/cf-ranges.txt`, `data/cf-ranges-v6.txt`), a preset, or `--custom-cidrs`.
2. **Probe** — TLS handshake (or TCP / HTTP) against each candidate, in parallel.
   Failures are filtered by latency, idle-hold and colo.
3. **Phase 2 (optional)** — verify survivors through a real xray tunnel using
   your own share URI, with optional DPI-bypass fragmentation.
4. **Export** — render the working set in the requested format.

### Repository layout

```
src/
  main.rs          thin binary entry point
  lib.rs           public facade
  cli.rs           clap types;  cli/scan_args.rs  flag -> config mapping
  api/             config, types and validation (ScanConfig::validate is the
                   single admission gate)
  engine/          the scan engine: driver, cdn, warp, phase2, plan, store,
                   neighbor, speed
  ranges/          Cloudflare range pool + refresh
  configs/         share-URI / subscription parsing
  probe.rs verify.rs inline_verify.rs   probing and verification
  export.rs        export registry (FORMATS) and renderers
  paths.rs         data dir, secret file permissions
  xray.rs dgst.rs  xray lifecycle, checksum grammar
tests/             integration tests, fixtures, proptest regressions
data/              pinned Cloudflare ranges, WARP pools, version pins
npm/cf-scanner/    npm wrapper package
specs/             hardening spec and its tickets
wix/               WiX template for the MSI installer
```

## Development

The toolchain is pinned to Rust 1.88 by `rust-toolchain.toml` (edition 2024).
Always pass `--locked`.

```bash
cargo build --all-targets --locked
cargo test --locked
cargo test --doc --locked
cargo clippy --all-targets --locked -- -D warnings
cargo fmt --check
cargo check --all-features --locked
cargo llvm-cov --all-targets --locked --lcov --output-path lcov.info --fail-under-lines 80
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full workflow, and
[AGENTS.md](AGENTS.md) for the architecture contracts that must not regress.

## Security

`cf-scanner` handles credentials: subscription URLs, share URIs, WARP private
keys. Secrets are written with owner-only permissions (`0600`/`0700` on Unix, a
protected DACL on Windows) and are never logged — errors are sanitized. See
[SECURITY.md](SECURITY.md) for reporting a vulnerability.

## License

MIT — see [LICENSE](LICENSE).
