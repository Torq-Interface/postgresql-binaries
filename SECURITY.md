# Security Policy

## Supported Versions

We provide PostgreSQL binaries aligned with [PostgreSQL's versioning policy](https://www.postgresql.org/support/versioning/).

| PostgreSQL Version | Supported          |
| ------------------ | ------------------ |
| 18.x               | ✅ Yes             |
| 17.x               | ✅ Yes             |
| 16.x               | ✅ Yes             |
| 15.x               | ✅ Yes             |
| 14.x               | ✅ Yes             |
| 13.x               | ⚠️ Limited (EOL Nov 2025) |
| < 13.x             | ❌ No              |

## Reporting a Vulnerability

If you discover a security vulnerability in our build pipeline or distribution process:

1. **DO NOT** open a public GitHub issue
2. Report security concerns via GitHub's private vulnerability reporting: https://github.com/Torq-Interface/postgresql-binaries/security/advisories/new
3. Include:
   - Description of the vulnerability
   - Steps to reproduce
   - Potential impact
   - Affected versions/platforms

**Response Time**: We aim to respond within 48 hours.

## Verifying Downloads

See [README.md](README.md#verifying-downloads) for instructions on verifying release authenticity using GitHub attestations.

## Security Measures

Our build pipeline includes:
- ✅ GPG verification of PostgreSQL source code
- ✅ SSL/TLS for all downloads
- ✅ Checksum verification for third-party dependencies
- ✅ Pinned dependencies (all packages version-locked)
- ✅ Cryptographic attestations (SLSA Build Level 2+)
- ✅ Non-root runtime users in Docker builds
- ✅ Automated testing on all platforms

## Supply Chain Security

- **Source Code Verification**: PostgreSQL source code verified using GPG signatures
- **Dependency Pinning**: All dependencies pinned to specific versions
- **Docker Images**: Base images pinned to cryptographic digests
- **GitHub Actions**: Actions pinned to commit SHAs
- **Build Attestations**: Every artifact includes cryptographic build provenance
- **Transparency**: Public Rekor log provides immutable audit trail

See [docs/DEPENDENCY-UPDATES.md](docs/DEPENDENCY-UPDATES.md) for dependency update procedures.

## Build Security

### Input Verification
- PostgreSQL source code: GPG signature verification
- Docker base images: Digest pinning (sha256)
- System packages: Version pinning for reproducibility
- GitHub Actions: Commit SHA pinning

### Build Process
- Isolated build environments (Docker containers, GitHub Actions runners)
- No privileged operations during compilation
- QEMU for cross-platform builds (version pinned)
- Automated testing before release

### Output Attestations
- SLSA Provenance v1.0 format
- Sigstore keyless signing
- Public transparency log (Rekor)
- Per-artifact attestations

## Known Limitations

### Windows Binaries
- **Source**: Downloaded from EnterpriseDB (not built from source)
- **Verification**: EnterpriseDB does not publish checksums for Windows binaries
- **Status**: This is a known security limitation
- **Future Work**: Build Windows binaries from source (tracked in issues)

## Compliance

This project implements security best practices aligned with:
- [SLSA Build Level 2+](https://slsa.dev/spec/v1.0/levels)
- [NIST SSDF](https://csrc.nist.gov/projects/ssdf) (Secure Software Development Framework)
- [OpenSSF Scorecard](https://github.com/ossf/scorecard) recommendations

## Security Contacts

For security-related questions or concerns:
- **Private Reports**: Use GitHub's private vulnerability reporting
- **General Questions**: Open a public GitHub issue (if not sensitive)
- **Documentation**: See README.md and docs/ directory

## Acknowledgments

We appreciate responsible disclosure of security vulnerabilities. Contributors who report valid security issues will be acknowledged in our release notes (unless they prefer to remain anonymous).
