# PostgreSQL Binaries

[![CI](https://github.com/Torq-Interface/postgresql-binaries/actions/workflows/ci.yml/badge.svg?branch=master)](https://github.com/Torq-Interface/postgresql-binaries/actions?query=workflow%3Aci+branch%3Amaster)
[![License](https://img.shields.io/github/license/Torq-Interface/postgresql-binaries)](./LICENSE)
[![Github All Releases](https://img.shields.io/github/downloads/Torq-Interface/postgresql-binaries/total.svg)]()

PostgreSQL binaries for Linux, MacOS and Windows; releases aligned with Rust [supported platforms](https://doc.rust-lang.org/nightly/rustc/platform-support.html).

---

## Choosing an installation package

Installation packages use the pattern `postgresql-<version>-<target>.<extension>`, where`<version>` is the
PostgreSQL version, `<target>` is the target triple for the platform, and `<extension>` is the archive file
extension.  To find the `<target>` triple for your platform run `rustc -vV` and look for the value of the
`host` field.  The target triple can be obtained programmatically in rust using the [target-triple](https://crates.io/crates/target-triple) crate.

## Versioning

This project uses a versioning scheme that is compatible with [PostgreSQL versioning](https://www.postgresql.org/support/versioning/).
The version comprises `<postgres major>.<postgres minor>.<release>`, where `<release>` is the release for
this project's build of a version of PostgreSQL.  New releases of this project will be made when new versions
of PostgreSQL are released or new builds of existing versions are required for bug fixes, new targets, etc.

## Verifying Downloads

All releases include cryptographic attestations proving authenticity and build provenance.

### Why Verify?

Verification ensures:
- Binary was built by our official GitHub Actions workflow
- Binary has not been tampered with after build
- Binary is traceable to exact source code commit
- Build process is auditable via public transparency log

### How to Verify

**Requirements**: [GitHub CLI](https://cli.github.com/) installed

**Verification command**:
```bash
# Download a release artifact
curl -LO https://github.com/Torq-Interface/postgresql-binaries/releases/download/17.6.0/postgresql-17.6.0-x86_64-unknown-linux-gnu.tar.gz

# Verify attestation
gh attestation verify postgresql-17.6.0-x86_64-unknown-linux-gnu.tar.gz \
  --owner Torq-Interface
```

**Successful verification output**:
```
✓ Verification succeeded!

  Repository: Torq-Interface/postgresql-binaries
  Workflow: .github/workflows/release.yml
  Commit: <commit-sha>
  Signed: <timestamp>
```

**What this proves**:
- ✅ Binary built from specific commit in this repository
- ✅ Built using official GitHub Actions workflow
- ✅ No modifications since build completed
- ✅ Immutable record in public transparency log (Rekor)

**Note**: Every platform build has its own attestation. Verify the specific platform you downloaded.

### Attestation Details

- **Format**: SLSA Provenance v1.0
- **Signing**: Sigstore (keyless signing via OIDC)
- **Transparency Log**: Public Rekor instance
- **Verification**: GitHub CLI or cosign
- **Coverage**: All platform builds (Linux, macOS, Windows)

For more information: https://docs.github.com/en/actions/security-guides/using-artifact-attestations

## Creating a New Release

### Single Version Release (Recommended)

For releasing a single PostgreSQL version, simply create and push a git tag:

1. **Determine the version number**: Use the format `<postgres major>.<postgres minor>.<release>`
   - Example: `17.6.0` for the first release of PostgreSQL 17.6
   - Example: `17.6.1` for a subsequent build update of PostgreSQL 17.6

2. **Create and push a git tag**:
   ```bash
   git tag 17.6.0
   git push origin 17.6.0
   ```

3. **Monitor the release workflow**:
   - The [release workflow](https://github.com/Torq-Interface/postgresql-binaries/actions/workflows/release.yml) will automatically trigger
   - A draft release will be created
   - Binaries will be built for all supported platforms (Linux, MacOS, Windows)
   - Archives and checksums will be uploaded to the release
   - The release will be automatically published when all builds complete successfully

4. **Verify the release**:
   - Check the [releases page](https://github.com/Torq-Interface/postgresql-binaries/releases) to confirm all assets were uploaded
   - Each platform should have `.tar.gz` archives with corresponding `.sha256` checksum files
   - Windows builds will additionally include `.zip` archives

**Note**: The workflow automatically strips the third version component when fetching PostgreSQL source code, as PostgreSQL uses 2-part versioning (e.g., tag `17.6.0` will build PostgreSQL version `17.6`).

### Bulk Release (Advanced)

For releasing multiple PostgreSQL versions simultaneously, use the `release.sh` script:

1. **Update the version list**: Edit `scripts/release.sh` and modify the `all_versions` array to include the PostgreSQL versions you want to release

2. **Create release notes**: Create a `release_notes.md` file in the root directory with your release notes

3. **Run the release script**:
   ```bash
   ./scripts/release.sh
   ```

The script will:
- Verify you're on the `master` branch with no uncommitted changes
- Check that your branch is up to date with `origin/master`
- Prevent re-releasing existing versions
- Append the release number (`.0`) to each version
- Create and push all tags with the release notes attached

**When to use bulk release**:
- Initial project setup when building binaries for all supported PostgreSQL versions
- After adding support for a new platform and backfilling all versions
- When releasing coordinated updates across multiple PostgreSQL versions

## License

PostgreSQL is covered under [The PostgreSQL License](https://opensource.org/licenses/postgresql).
