<p align="center"><img src="logo.png" alt="prov-engine-js logo" width="200"></p>

# prov-engine-js

> Part of [MCP Tool Shop](https://mcptoolshop.com)

**A minimal, zero-dependency Node.js provenance engine implementing the prov-spec standard.**

[![npm version](https://img.shields.io/npm/v/@mcptoolshop/prov-engine-js)](https://www.npmjs.com/package/@mcptoolshop/prov-engine-js)
[![license](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![zero dependencies](https://img.shields.io/badge/dependencies-0-brightgreen)]()
[![node](https://img.shields.io/badge/node-18%2B-blue)]()
[![prov-spec L1](https://img.shields.io/badge/prov--spec-L1%20Integrity-blue)](https://github.com/prov-spec/prov-spec)

---

## At a Glance

- **Zero dependencies** -- uses only Node.js built-ins (`node:fs`, `node:crypto`, `node:process`)
- **Single-file engine** -- the entire implementation lives in `prov-engine.js`
- **CLI + programmatic** -- run from the command line or import into your own code
- **prov-spec L1 conformant** -- passes all Level 1 (Integrity) test vectors
- **JCS-subset canonicalization** -- deterministic JSON serialization per prov-spec Section 6

---

## Install

```bash
# Add to your project
pnpm add @mcptoolshop/prov-engine-js

# Or run directly without installing
npx @mcptoolshop/prov-engine-js describe
```

You can also clone the repo and run the engine directly:

```bash
git clone https://github.com/mcp-tool-shop-org/prov-engine-js.git
cd prov-engine-js
node prov-engine.js describe
```

---

## CLI Commands

The engine exposes five commands. All output is JSON written to stdout.

### `describe` -- Print capability manifest

```bash
npx @mcptoolshop/prov-engine-js describe
```

```json
{
  "schema": "prov-capabilities@v0.1",
  "engine": {
    "name": "prov-engine-js",
    "version": "0.1.0",
    "vendor": "prov-spec",
    "repo": "https://github.com/prov-spec/prov-engine-js",
    "license": "MIT"
  },
  "implements": [
    "adapter.wrap.envelope_v0_1",
    "integrity.digest.sha256"
  ],
  "conformance_level": "fully-conformant"
}
```

### `digest <input.json>` -- Compute canonical form and SHA-256 digest

```bash
echo '{"b":2,"a":1}' > input.json
npx @mcptoolshop/prov-engine-js digest input.json
```

```json
{
  "canonical_form": "{\"a\":1,\"b\":2}",
  "digest": {
    "alg": "sha256",
    "value": "abd8d7fa4bab05cdd8da39bee28237e3b2c9cb08ccfc73e0af3e5a6f17eaee5a"
  }
}
```

### `wrap <payload.json>` -- Wrap payload in an MCP envelope

```bash
echo '{"tool":"example","result":"ok"}' > payload.json
npx @mcptoolshop/prov-engine-js wrap payload.json
```

```json
{
  "schema_version": "mcp.envelope.v0.1",
  "result": {
    "tool": "example",
    "result": "ok"
  }
}
```

If the input is already an envelope (`schema_version` equals `mcp.envelope.v0.1`), it is passed through unchanged (no double-wrapping).

### `verify-digest <artifact.json>` -- Verify a digest claim

The input file must contain a `content` field and a `digest` field with `alg` and `value`:

```bash
cat > artifact.json << 'EOF'
{
  "content": {"a": 1, "b": 2},
  "digest": {
    "alg": "sha256",
    "value": "abd8d7fa4bab05cdd8da39bee28237e3b2c9cb08ccfc73e0af3e5a6f17eaee5a"
  }
}
EOF
npx @mcptoolshop/prov-engine-js verify-digest artifact.json
# Exit code 0 = valid, exit code 1 = mismatch
echo $?  # 0
```

### `check-vector <vector-dir>` -- Run a prov-spec test vector

```bash
npx @mcptoolshop/prov-engine-js check-vector ../prov-spec/spec/vectors/integrity.digest.sha256
# PASS: integrity.digest.sha256 vector

npx @mcptoolshop/prov-engine-js check-vector ../prov-spec/spec/vectors/adapter.wrap.envelope_v0_1
# PASS: adapter.wrap.envelope_v0_1 vector
```

The vector directory must contain `input.json` and `expected.json`. The engine auto-detects the vector type from the expected output shape.

---

## Programmatic Usage

The engine is an ES module. You can import its internals for use in your own code:

```js
import { createHash } from "node:crypto";
import { readFileSync } from "node:fs";

// -- Canonicalization (inline, since not exported yet) --
function canonicalize(value) {
  if (value === null) return "null";
  if (typeof value === "boolean") return value ? "true" : "false";
  if (typeof value === "string") return JSON.stringify(value);
  if (typeof value === "number") {
    if (!Number.isFinite(value)) throw new Error("Non-finite numbers not allowed");
    return JSON.stringify(value);
  }
  if (Array.isArray(value)) return "[" + value.map(canonicalize).join(",") + "]";
  if (typeof value === "object") {
    const keys = Object.keys(value).sort();
    return "{" + keys.map(k => JSON.stringify(k) + ":" + canonicalize(value[k])).join(",") + "}";
  }
  throw new Error(`Non-JSON value type: ${typeof value}`);
}

// Compute a digest
const payload = { tool: "demo", version: 1 };
const canonical = canonicalize(payload);
const hash = createHash("sha256").update(canonical, "utf8").digest("hex");

console.log("Canonical:", canonical);
console.log("SHA-256:  ", hash);

// Wrap in an MCP envelope
const envelope = {
  schema_version: "mcp.envelope.v0.1",
  result: payload,
};
console.log("Envelope: ", JSON.stringify(envelope, null, 2));
```

> **Tip:** A future release will export `canonicalize`, `computeDigest`, and `wrapEnvelope` as named exports so you can `import { canonicalize } from "@mcptoolshop/prov-engine-js"` directly.

---

## Methods

| Method | Description |
|--------|-------------|
| `integrity.digest.sha256` | Canonicalize JSON per prov-spec Section 6, then compute SHA-256 over the UTF-8 bytes. Returns `{ alg: "sha256", value: "<hex>" }`. |
| `adapter.wrap.envelope_v0_1` | Wrap any JSON payload in `{ schema_version: "mcp.envelope.v0.1", result: <payload> }`. Already-wrapped envelopes pass through unchanged. |

---

## How Canonicalization Works

prov-spec requires deterministic JSON serialization so that the same logical object always produces the same byte sequence (and therefore the same digest). This engine implements a JCS-subset canonicalization per prov-spec Section 6:

1. **Sorted keys** -- Object keys are sorted lexicographically by Unicode code point order.
2. **No whitespace** -- No spaces or newlines between tokens. Separators are `,` and `:` only.
3. **Number normalization** -- No leading zeros, no trailing zeros after decimal point, no positive sign, `-0` becomes `0`.
4. **Minimal string escaping** -- Only required escape sequences are emitted.
5. **UTF-8 encoding** -- The canonical string is encoded as UTF-8 before hashing.

Example: `{"b": 2, "a": 1}` canonicalizes to `{"a":1,"b":2}`.

---

## Conformance

This engine declares **fully-conformant** status for **Level 1 (Integrity)** of the prov-spec standard.

- Passes all `integrity.digest.sha256` test vectors
- Passes all `adapter.wrap.envelope_v0_1` test vectors
- Zero known deviations

See `prov-capabilities.json` for the full capability manifest.

---

## Docs

| Document | Description |
|----------|-------------|
| [HANDBOOK.md](HANDBOOK.md) | Deep-dive guide: provenance concepts, prov-spec mechanics, integration patterns, architecture, FAQ |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute, development workflow, design principles |
| [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) | Community standards |
| [CHANGELOG.md](CHANGELOG.md) | Release history (Keep a Changelog format) |

---

## License

[MIT](LICENSE)
