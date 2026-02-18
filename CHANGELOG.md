# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - 2026-02-17

### Added

- Comprehensive README with badges, install instructions, CLI examples, programmatic usage, and docs table
- HANDBOOK.md -- deep-dive guide covering provenance concepts, prov-spec mechanics, canonical JSON internals, integration patterns, architecture, and FAQ
- CHANGELOG.md (this file)

### Changed

- README rewritten from scratch with full CLI command reference, "At a Glance" section, methods table, and canonicalization explainer

## [0.2.1] - 2026-02-01

### Fixed

- Repository URLs updated from `mcp-tool-shop` to `mcp-tool-shop-org` to reflect canonical org ownership

### Changed

- npm scope migrated from `@mcp-tool-shop` to `@mcptoolshop`
- Added `publishConfig` with public access to package.json

## [0.2.0] - 2026-01-30

### Changed

- Package published to npm as `@mcp-tool-shop/prov-engine-js` (scoped package)
- Added MCP Tool Shop catalog link to README
- Added project documentation (CONTRIBUTING.md, CODE_OF_CONDUCT.md)
- Added CI workflow (GitHub Actions, Node.js 18/20/22 matrix)

## [0.1.0] - 2026-01-28

### Added

- Initial release
- Single-file provenance engine (`prov-engine.js`) with zero dependencies
- `integrity.digest.sha256` method -- canonical JSON + SHA-256 digest
- `adapter.wrap.envelope_v0_1` method -- MCP envelope wrapping with pass-through rule
- CLI commands: `describe`, `digest`, `wrap`, `verify-digest`, `check-vector`
- `prov-capabilities.json` capability manifest
- JCS-subset canonicalization per prov-spec Section 6
- prov-spec Level 1 (Integrity) full conformance

[0.3.0]: https://github.com/mcp-tool-shop-org/prov-engine-js/compare/v0.2.1...v0.3.0
[0.2.1]: https://github.com/mcp-tool-shop-org/prov-engine-js/compare/v0.2.0...v0.2.1
[0.2.0]: https://github.com/mcp-tool-shop-org/prov-engine-js/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/mcp-tool-shop-org/prov-engine-js/releases/tag/v0.1.0
