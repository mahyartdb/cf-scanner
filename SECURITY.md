# Security Policy

## Reporting a vulnerability

Please **do not open a public issue** for a security problem.

Report it privately via
[GitHub Security Advisories](https://github.com/QMahyar/cf-scanner/security/advisories/new)
("Report a vulnerability"). Include the affected version, the platform, a
reproduction, and the impact you observed.

You can expect an acknowledgement within a few days. If no fix is warranted,
you will get a clear explanation of why.

## What `cf-scanner` handles

This tool deals with credentials, so treat it accordingly:

- **Subscription URLs** (`check-sub <URL>`) — these are bearer credentials. Pass
  them as a shell argument only if you accept them landing in your shell
  history and process list.
- **Share URIs** (`vless://`, `vmess://`, `trojan://`, `ss://`) — they embed
  UUIDs, passwords and tokens.
- **WARP identities** (`warp-config generate`) — contain a private key.

The tool stores these in its data dir with owner-only permissions (`0600` /
`0700` on Unix, a protected DACL on Windows) and never logs them. Errors are
sanitized so that credentials do not leak through error messages.

## Handling guidance

- Keep the data dir out of shared locations. It defaults to the OS
  user-data directory; `CF_SCANNER_DATA_DIR` overrides it.
- Exports (`--export`, `--export-live`) contain working endpoints and, for
  phase-2 runs, credentials. Treat them as sensitive files.
- `--retry-last` saves the scan configuration but **never** saves phase-2
  configs or WARP keys. You still have to re-supply them.
- The pinned xray binary is verified by SHA-256 against the upstream `.dgst`
  sidecar before it is used; the checksum must parse as `SHA2-256= <64 hex>`.

## Scope

Reports about the upstream Cloudflare, Xray or Cloudflare WARP services are
not in scope for this repository — please report those to their respective
maintainers.
