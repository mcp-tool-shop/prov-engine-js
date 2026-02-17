# prov-engine-js

> Part of [MCP Tool Shop](https://mcptoolshop.com)


**A minimal, zero-dependency Node.js provenance engine for prov-spec interop.**

[![prov-spec L1](https://img.shields.io/badge/prov--spec-L1%20Integrity-blue)](https://github.com/prov-spec/prov-spec)

## Purpose

This engine exists to prove that prov-spec is language-agnostic. It implements the same methods as the Python reference validator, passes the same test vectors, and shares no code.

## Requirements

- Node.js 18+
- No dependencies

## Usage

```bash
# Print capability manifest
node prov-engine.js describe

# Compute canonical form and SHA-256 digest
node prov-engine.js digest input.json

# Wrap payload in mcp.envelope.v0.1
node prov-engine.js wrap payload.json

# Verify a digest claim
node prov-engine.js verify-digest artifact.json

# Run against a prov-spec test vector
node prov-engine.js check-vector ../prov-spec/spec/vectors/integrity.digest.sha256
```

## Implemented Methods

| Method | Description |
|--------|-------------|
| `integrity.digest.sha256` | Canonical JSON + SHA-256 digest |
| `adapter.wrap.envelope_v0_1` | Wrap payload in MCP envelope |

## Test Vectors

```bash
# Run all vectors
node prov-engine.js check-vector ../prov-spec/spec/vectors/integrity.digest.sha256
node prov-engine.js check-vector ../prov-spec/spec/vectors/adapter.wrap.envelope_v0_1
```

Expected output:

```
PASS: integrity.digest.sha256 vector
PASS: adapter.wrap.envelope_v0_1 vector
```

## Conformance

This engine declares `fully-conformant` status for Level 1 (Integrity).

See `prov-capabilities.json` for the full capability manifest.

## License

MIT
