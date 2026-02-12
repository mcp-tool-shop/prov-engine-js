# Contributing to prov-engine-js

Thank you for your interest in contributing to prov-engine-js! This is a minimal, zero-dependency Node.js provenance engine that demonstrates prov-spec interoperability.

## How to Contribute

### Reporting Issues

If you find a bug or have a suggestion:

1. Check if the issue already exists in [GitHub Issues](https://github.com/mcp-tool-shop-org/prov-engine-js/issues)
2. If not, create a new issue with:
   - A clear, descriptive title
   - Steps to reproduce (for bugs)
   - Expected vs. actual behavior
   - Your environment (Node version, OS)

### Contributing Code

1. **Fork the repository** and create a branch from `main`
2. **Make your changes**
   - Follow the existing code style
   - Maintain zero-dependency requirement
   - Ensure compatibility with Node.js 18+
   - Add test vectors if adding new methods
3. **Test your changes**
   ```bash
   npm test
   ```
4. **Commit your changes**
   - Use clear, descriptive commit messages
   - Reference issue numbers when applicable
5. **Submit a pull request**
   - Describe what your PR does and why
   - Link to related issues

### Development Workflow

```bash
# Run all test vectors
npm test

# Test specific functionality
node prov-engine.js describe
node prov-engine.js digest input.json
node prov-engine.js wrap payload.json
node prov-engine.js verify-digest artifact.json

# Check against prov-spec vectors (requires prov-spec repo)
node prov-engine.js check-vector ../prov-spec/spec/vectors/integrity.digest.sha256
```

### Adding New prov-spec Methods

1. Implement the method in `prov-engine.js`
2. Update `prov-capabilities.json` with the new method
3. Add corresponding test vectors (if available from prov-spec)
4. Update README.md with method documentation
5. Ensure zero dependencies are maintained

### Design Principles

- **Zero dependencies** - Only use Node.js built-ins
- **Minimal** - Keep the implementation simple and focused
- **Interoperable** - Must pass prov-spec test vectors
- **Deterministic** - Same input produces same output
- **Conformant** - Follow prov-spec exactly

### Code Style

- Use ES modules (`import`/`export`)
- Use modern JavaScript (Node 18+ features)
- Prefer `const` over `let`
- Use descriptive variable names
- Keep functions small and focused

### Testing

All implementations must:
- Pass prov-spec test vectors
- Produce deterministic output
- Match the Python reference implementation behavior

## prov-spec Conformance

This engine targets **Level 1 (Integrity)** conformance with prov-spec.

Currently implemented:
- `integrity.digest.sha256` - Canonical JSON + SHA-256
- `adapter.wrap.envelope_v0_1` - MCP envelope wrapping

## Code of Conduct

Please note that this project is released with a [Code of Conduct](CODE_OF_CONDUCT.md). By participating, you agree to abide by its terms.

## Questions?

Open an issue or start a discussion. We're here to help!
