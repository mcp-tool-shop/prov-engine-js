# prov-engine-js Handbook

A comprehensive guide to provenance, prov-spec, and this engine's internals.

---

## Table of Contents

1. [What Is Provenance?](#what-is-provenance)
2. [Why Provenance Matters for AI Tools](#why-provenance-matters-for-ai-tools)
3. [How prov-spec Works](#how-prov-spec-works)
4. [Deep Dive: Canonical JSON](#deep-dive-canonical-json)
5. [Deep Dive: SHA-256 Digest Computation](#deep-dive-sha-256-digest-computation)
6. [Deep Dive: MCP Envelope Wrapping](#deep-dive-mcp-envelope-wrapping)
7. [Integration Patterns](#integration-patterns)
8. [Test Vector Format](#test-vector-format)
9. [Conformance Levels Explained](#conformance-levels-explained)
10. [Architecture](#architecture)
11. [FAQ](#faq)

---

## What Is Provenance?

Provenance is the record of where something came from and what happened to it. In software, provenance answers questions like:

- **Origin** -- Who produced this artifact? Which tool, which version?
- **Integrity** -- Has this artifact been tampered with since it was created?
- **Chain of custody** -- What transformations were applied, and in what order?

For data flowing through automated pipelines, provenance provides a verifiable trail. Without it, consumers must blindly trust every intermediary.

---

## Why Provenance Matters for AI Tools

AI tool ecosystems (like the Model Context Protocol) involve chains of tool calls where one tool's output feeds into the next. This creates specific trust problems:

**Tampering detection.** If a tool returns a result and a downstream system modifies it before presenting it to the user, the user cannot tell. A cryptographic digest attached at the source lets any party re-compute the digest and detect changes.

**Auditability.** When an AI agent calls five tools in sequence, provenance records let you reconstruct exactly what each tool returned. This matters for debugging, compliance, and dispute resolution.

**Interoperability.** If every tool vendor invents their own integrity scheme, consumers need N different verification libraries. prov-spec standardizes the approach so that one engine (like this one) can verify artifacts from any conformant producer.

**Reproducibility.** Deterministic canonicalization means two independent implementations that receive the same logical JSON will compute the same digest. This is the foundation of cross-language, cross-vendor trust.

---

## How prov-spec Works

prov-spec is a layered specification for tool-output provenance. It defines:

### Levels

| Level | Name | What It Covers |
|-------|------|----------------|
| L1 | Integrity | Canonical JSON + cryptographic digests. Proves content has not changed. |
| L2 | Attribution | Signatures and identity. Proves who produced the content. |
| L3 | Lineage | Dependency chains and transformation records. Proves how content was derived. |

This engine implements **Level 1 (Integrity)** fully.

### Capabilities

Each engine publishes a capability manifest (`prov-capabilities.json`) declaring:

- Which methods it implements (e.g., `integrity.digest.sha256`)
- Its conformance level (`fully-conformant`, `partially-conformant`)
- Any known deviations from the spec
- Which test vectors it has validated against

### Test Vectors

prov-spec ships test vectors: pairs of `input.json` and `expected.json` files. Any conformant engine must produce the expected output for each input. This is how cross-language interoperability is verified -- the Python reference implementation and this Node.js engine must agree on every vector.

### Methods

A method is a named operation with defined input/output schemas. Currently defined:

- `integrity.digest.sha256` -- Takes any JSON value, produces `{ canonical_form, digest: { alg, value } }`
- `adapter.wrap.envelope_v0_1` -- Takes any JSON payload, produces `{ schema_version: "mcp.envelope.v0.1", result: <payload> }`

---

## Deep Dive: Canonical JSON

### The Problem

JSON is not deterministic by default. These three strings represent the same logical object:

```json
{"b": 2, "a": 1}
{"a":1,"b":2}
{ "a" : 1 , "b" : 2 }
```

If you hash them as raw bytes, you get three different digests for the same data. Canonicalization solves this by defining a single, deterministic serialization for any JSON value.

### The Rules (prov-spec Section 6)

prov-engine-js implements a JCS-subset (RFC 8785 compatible) canonicalization:

1. **Encoding**: UTF-8
2. **Object keys**: Sorted lexicographically by Unicode code point order (the default `Array.sort()` behavior in JavaScript)
3. **Whitespace**: None. No spaces after `:` or `,`, no newlines, no indentation
4. **Number format**:
   - No leading zeros (`01` is not valid JSON anyway)
   - No trailing zeros after a decimal point (`1.0` becomes `1`)
   - No positive sign (`+1` becomes `1`)
   - Negative zero becomes `0` (`-0` becomes `0`)
   - Non-finite values (`Infinity`, `NaN`) are rejected
5. **String escaping**: Minimal -- only the escapes required by JSON (`\"`, `\\`, `\n`, `\r`, `\t`, `\b`, `\f`, and `\uXXXX` for control characters)
6. **Separators**: `,` between array elements and object members, `:` between keys and values
7. **Recursion**: Arrays and nested objects are canonicalized recursively

### Example Walkthrough

Input (parsed from any formatting):

```json
{
  "name": "example",
  "version": 2,
  "tags": ["beta", "alpha"],
  "metadata": {
    "z_field": true,
    "a_field": null
  }
}
```

Canonical output:

```
{"metadata":{"a_field":null,"z_field":true},"name":"example","tags":["beta","alpha"],"version":2}
```

Note:
- Top-level keys are sorted: `metadata` before `name` before `tags` before `version`
- Nested object keys are sorted: `a_field` before `z_field`
- Array order is preserved (arrays are ordered; objects are not)
- No whitespace anywhere

### Why Not Just `JSON.stringify(obj, Object.keys(obj).sort())`?

`JSON.stringify` with a replacer does not recursively sort nested objects. It also adds whitespace if you pass an indent argument. The canonicalize function in this engine handles all nesting levels and produces strictly minimal output.

---

## Deep Dive: SHA-256 Digest Computation

### The Pipeline

```
JSON value
  --> canonicalize(value)     : produces a deterministic UTF-8 string
  --> encode as UTF-8 bytes   : Node.js crypto handles this via the "utf8" encoding flag
  --> SHA-256 hash            : produces 32 bytes
  --> hex-encode              : produces 64-character lowercase hex string
```

### In Code

The engine's digest pipeline is two functions:

```js
function sha256Hex(utf8String) {
  const hash = crypto.createHash("sha256");
  hash.update(utf8String, "utf8");
  return hash.digest("hex");
}

function computeDigest(content) {
  const canonical = canonicalize(content);
  const hex = sha256Hex(canonical);
  return { alg: "sha256", value: hex };
}
```

### Output Format

The `digest` command outputs:

```json
{
  "canonical_form": "{\"a\":1,\"b\":2}",
  "digest": {
    "alg": "sha256",
    "value": "abd8d7fa4bab05cdd8da39bee28237e3b2c9cb08ccfc73e0af3e5a6f17eaee5a"
  }
}
```

The `canonical_form` field is included so you can inspect exactly what bytes were hashed.

### Verification

To verify a digest, re-canonicalize the content and re-hash. If the hex strings match, the content has not been modified. The `verify-digest` command does exactly this: it reads a `content` field and a `digest` field, re-computes, and compares. Exit code 0 means valid; exit code 1 means mismatch.

---

## Deep Dive: MCP Envelope Wrapping

### What Is an MCP Envelope?

The Model Context Protocol (MCP) defines a standard envelope format for tool outputs. An envelope wraps any payload in a versioned container:

```json
{
  "schema_version": "mcp.envelope.v0.1",
  "result": <any JSON payload>
}
```

This is useful because it gives every tool output a uniform shape, making it easier for consuming systems to route, validate, and log.

### Wrapping Behavior

The `wrap` command takes a JSON file and wraps it:

- If the input is **not** already an envelope, it becomes the `result` field of a new envelope.
- If the input **is** already an envelope (its `schema_version` equals `mcp.envelope.v0.1`), it passes through unchanged. This prevents double-wrapping.

### Pass-Through Rule

The pass-through rule is important for idempotency. If a tool wraps its output and then a pipeline stage wraps it again, you would get nested envelopes. The pass-through check prevents this:

```js
if (payload.schema_version === "mcp.envelope.v0.1") {
  // Already wrapped -- return as-is
  return payload;
}
```

---

## Integration Patterns

### Using in CI

Add a digest verification step to your CI pipeline to ensure artifacts have not been tampered with between build and deploy:

```yaml
# .github/workflows/verify.yml
- name: Verify artifact integrity
  run: |
    npx @mcptoolshop/prov-engine-js verify-digest build-output.json
```

You can also compute digests at build time and store them alongside your artifacts:

```yaml
- name: Compute provenance digest
  run: |
    npx @mcptoolshop/prov-engine-js digest dist/manifest.json > dist/manifest.digest.json
```

### Using in MCP Tools

If you are building an MCP tool server, wrap your tool outputs in envelopes before returning them:

```js
import { execSync } from "node:child_process";
import { writeFileSync, readFileSync, unlinkSync } from "node:fs";

function wrapToolOutput(result) {
  const tmpPath = `/tmp/prov-payload-${Date.now()}.json`;
  writeFileSync(tmpPath, JSON.stringify(result));
  const output = execSync(`npx @mcptoolshop/prov-engine-js wrap ${tmpPath}`, {
    encoding: "utf8",
  });
  unlinkSync(tmpPath);
  return JSON.parse(output);
}
```

### Using as a Library

For tighter integration, copy the `canonicalize` function into your codebase (it is self-contained and has no dependencies) and use `node:crypto` directly:

```js
import { createHash } from "node:crypto";

// Copy canonicalize() from prov-engine.js -- it is ~20 lines

function provDigest(payload) {
  const canonical = canonicalize(payload);
  const hash = createHash("sha256").update(canonical, "utf8").digest("hex");
  return {
    canonical_form: canonical,
    digest: { alg: "sha256", value: hash },
  };
}
```

### Using in a Test Suite

Run conformance checks as part of your project's test suite:

```js
import { execSync } from "node:child_process";
import assert from "node:assert";

const vectorDirs = [
  "../prov-spec/spec/vectors/integrity.digest.sha256",
  "../prov-spec/spec/vectors/adapter.wrap.envelope_v0_1",
];

for (const dir of vectorDirs) {
  const output = execSync(`node prov-engine.js check-vector ${dir}`, {
    encoding: "utf8",
  });
  assert(output.startsWith("PASS"), `Vector failed: ${dir}`);
}
```

---

## Test Vector Format

prov-spec test vectors are directories containing two files:

```
vectors/
  integrity.digest.sha256/
    input.json        # The JSON value to process
    expected.json     # The expected output
  adapter.wrap.envelope_v0_1/
    input.json
    expected.json
```

### `integrity.digest.sha256` Vector

**input.json**: Any valid JSON value.

**expected.json**:
```json
{
  "canonical_form": "<canonical JSON string>",
  "digest": {
    "alg": "sha256",
    "value": "<64-char lowercase hex>"
  }
}
```

### `adapter.wrap.envelope_v0_1` Vector

**input.json**: Any valid JSON payload.

**expected.json**:
```json
{
  "schema_version": "mcp.envelope.v0.1",
  "result": <same as input.json>
}
```

### Adding New Test Vectors

1. Create a new directory under `spec/vectors/<method-name>/`.
2. Add `input.json` with your test input.
3. Use a known-good implementation to generate `expected.json`.
4. Run `check-vector` against the new directory to verify.

The engine auto-detects the vector type from the shape of `expected.json`:
- If it has `canonical_form` and `digest` fields, it is treated as an `integrity.digest.sha256` vector.
- If it has `schema_version` equal to `mcp.envelope.v0.1`, it is treated as an `adapter.wrap.envelope_v0_1` vector.

---

## Conformance Levels Explained

prov-spec defines two conformance statuses:

### Fully Conformant

An engine is **fully conformant** at a given level if:

- It implements all required methods for that level
- It passes all test vectors for those methods
- It reports zero known deviations

prov-engine-js is fully conformant at Level 1 (Integrity).

### Partially Conformant

An engine is **partially conformant** if:

- It implements some but not all methods for a level, or
- It passes most but not all vectors, or
- It reports known deviations (documented edge cases where behavior differs from the spec)

### Capability Manifest

Every engine publishes a `prov-capabilities.json` file declaring its conformance:

```json
{
  "schema": "prov-capabilities@v0.1",
  "engine": {
    "name": "prov-engine-js",
    "version": "0.1.0"
  },
  "implements": [
    "adapter.wrap.envelope_v0_1",
    "integrity.digest.sha256"
  ],
  "conformance_level": "fully-conformant",
  "test_vectors_validated": [
    "integrity.digest.sha256",
    "adapter.wrap.envelope_v0_1"
  ],
  "known_deviations": []
}
```

Consumers can read this manifest to determine what an engine supports before calling it.

---

## Architecture

### Single-File Design

The entire engine is a single file: `prov-engine.js`. This is intentional:

- **Auditability** -- You can read the whole implementation in under 5 minutes. For a security-relevant tool, this matters.
- **Portability** -- Copy one file into any Node.js project. No `node_modules`, no build step.
- **Transparency** -- No transitive dependencies to audit. What you see is what runs.

### No Dependencies

The engine uses three Node.js built-in modules:

| Module | Purpose |
|--------|---------|
| `node:fs` | Read input JSON files from disk |
| `node:crypto` | SHA-256 hash computation |
| `node:process` | CLI argument parsing, exit codes, stdout/stderr |

There are zero npm dependencies. This is a hard design constraint, not a temporary shortcut. Provenance tooling must be auditable, and every dependency is an audit surface.

### Code Organization

The file is organized into four sections:

1. **Utilities** -- `die()`, `readJsonFile()`, `writeJson()` -- basic I/O helpers
2. **Canonical JSON** -- `canonicalize()` -- the deterministic serializer
3. **Digest computation** -- `sha256Hex()`, `computeDigest()` -- hash pipeline
4. **Commands** -- `cmdDescribe()`, `cmdDigest()`, `cmdWrap()`, `cmdVerifyDigest()`, `cmdCheckVector()` -- CLI entry points

### Data Flow

```
CLI args
  --> main() parses command name and file path
  --> Command function reads JSON from disk
  --> Core functions (canonicalize, computeDigest) process the data
  --> Result is written to stdout as formatted JSON
  --> Exit code signals success (0) or failure (1)
```

---

## FAQ

### Is this a replacement for the Python prov-spec validator?

No. The Python reference implementation is the canonical validator maintained by the prov-spec project. This engine exists to prove that prov-spec is language-agnostic -- a completely independent implementation in a different language that passes the same test vectors.

### Can I use this in production?

Yes. The engine is stable, tested across Node.js 18/20/22, and has zero dependencies. It is suitable for CI pipelines, MCP tool servers, and anywhere you need deterministic JSON digests.

### Why JCS-subset and not full JCS (RFC 8785)?

prov-spec Section 6 defines a canonicalization that is compatible with JCS but does not require the full spec. The subset covers the cases that matter for tool-output provenance: sorted keys, minimal whitespace, normalized numbers. The engine does not need to handle exotic Unicode normalization or I-JSON edge cases that the full JCS spec addresses.

### Does this handle binary data?

No. The engine operates on JSON values. If you need to digest binary artifacts, hash the raw bytes separately. prov-spec's integrity methods are designed for structured JSON payloads.

### What happens if I pass invalid JSON?

The engine calls `JSON.parse()` on the input file. If parsing fails, Node.js throws a `SyntaxError` and the process exits with code 1.

### How do I verify a digest from a different engine?

Use the `verify-digest` command. Create a JSON file with `content` (the original payload) and `digest` (the claimed `{ alg, value }`), then run:

```bash
npx @mcptoolshop/prov-engine-js verify-digest artifact.json
```

If the digest matches, exit code is 0. If not, the engine prints a mismatch message and exits with code 1.

### Will this engine support Level 2 (Attribution) or Level 3 (Lineage)?

Not in the current scope. This engine targets Level 1 completeness. Higher levels would require signature infrastructure and dependency tracking, which go beyond the single-file zero-dependency design.

### Can I use this with CommonJS (`require`)?

No. The package is ESM-only (`"type": "module"` in package.json). Use `import` or dynamic `import()`.

### Where are the test vectors?

Test vectors live in the prov-spec repository, not in this repo. Clone [prov-spec](https://github.com/prov-spec/prov-spec) alongside this engine and point `check-vector` at the vectors directory.
