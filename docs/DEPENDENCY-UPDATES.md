# Dependency Update Process

This project pins all dependencies for security and reproducibility. This document describes how to update pinned versions.

## Update Schedule

- **Annual review** of dependency versions
- **Immediate updates** for security vulnerabilities
- **Test all changes** before merging

## Updating Alpine Packages (MUSL Builds)

**File**: [dockerfiles/Dockerfile.linux-musl](../dockerfiles/Dockerfile.linux-musl)

### Get Current Versions

```bash
docker run --rm alpine:3.19.0 sh -c 'apk update >/dev/null 2>&1 && apk list bash bison clang16 dpkg dpkg-dev e2fsprogs-dev flex g++ gcc gettext-dev git gnupg icu-dev krb5-dev libc-dev libxml2-dev libxslt-dev linux-headers llvm16-dev lz4 lz4-dev make musl-dev openssl-dev ossp-uuid-dev patchelf perl perl-dev perl-ipc-run perl-utils python3 python3-dev readline-dev util-linux-dev zlib-dev zstd-dev 2>/dev/null | grep -E "^\w" | sort'
```

### Update Dockerfile

1. Update package versions in Dockerfile
2. Update "Last updated" comment with current date
3. Test build: `docker build -f dockerfiles/Dockerfile.linux-musl --build-arg POSTGRESQL_VERSION=18.0.0 .`
4. Commit changes

## Updating Debian Packages (GNU Builds)

**File**: [dockerfiles/Dockerfile.linux-gnu](../dockerfiles/Dockerfile.linux-gnu)

### Get Current Versions

```bash
docker run --rm debian:12.4 sh -c 'apt-get update >/dev/null 2>&1 && for pkg in bison build-essential clang-16 flex fop gettext git gnupg libicu-dev libkrb5-dev liblz4-dev libossp-uuid-dev libperl-dev libreadline-dev libssl-dev libxml2-dev libxml2-utils libxslt1-dev libzstd-dev llvm-16 lz4 make openssl patchelf perl pkg-config python3 python3-dev wget xsltproc zlib1g-dev zstd; do version=$(apt-cache policy $pkg 2>/dev/null | grep "Candidate:" | awk "{print \$2}"); echo "$pkg=$version"; done'
```

### Update Dockerfile

1. Update package versions in Dockerfile
2. Update "Last updated" comment with current date
3. Test build: `docker build -f dockerfiles/Dockerfile.linux-gnu --build-arg POSTGRESQL_VERSION=18.0.0 .`
4. Commit changes

## Updating Homebrew Packages (macOS)

**File**: [.github/workflows/build.yml](.github/workflows/build.yml) (macOS build section)

### Current Status

⚠️ **Action Required**: Homebrew packages are not yet pinned. The macOS build currently installs latest versions.

### Recommended Approach

Create a Brewfile for version locking:

```yaml
- name: Create Brewfile (MacOS)
  if: ${{ startsWith(matrix.id, 'macos-') }}
  run: |
    cat > Brewfile <<'EOF'
    # PostgreSQL build dependencies
    # Last updated: YYYY-MM-DD
    brew "fop"
    brew "gettext"
    brew "icu4c"
    brew "lld"
    brew "llvm"
    brew "lz4"
    brew "openssl@3"
    brew "readline"
    brew "xz"
    brew "zstd"
    EOF

    brew bundle install --file=Brewfile
    brew bundle dump --file=Brewfile.lock.json --force
```

Then commit the generated `Brewfile.lock.json` to the repository and use it in subsequent builds.

## Updating Docker Base Images

**Files**:
- [dockerfiles/Dockerfile.linux-musl](../dockerfiles/Dockerfile.linux-musl)
- [dockerfiles/Dockerfile.linux-gnu](../dockerfiles/Dockerfile.linux-gnu)

### Get Image Digest

```bash
# For Alpine
docker pull alpine:3.19.0
docker inspect alpine:3.19.0 | grep "Id"
# Output: sha256:51b67269f354137895d43f3b3d810bfacd3945438e94dc5ac55fdac340352f48

# For Debian
docker pull debian:12.4
docker inspect debian:12.4 | grep "Id"
# Output: sha256:79becb70a6247d277b59c09ca340bbe0349af6aacb5afa90ec349528b53ce2c9
```

### Update Dockerfile

Replace `FROM alpine:3.19.0@sha256:OLD_DIGEST` with new digest.

## Updating tonistiigi/binfmt

**File**: [.github/workflows/build.yml](.github/workflows/build.yml)

### Get Latest Version

1. Check releases: https://github.com/tonistiigi/binfmt/releases
2. Pull image and get digest:

```bash
docker pull tonistiigi/binfmt:qemu-vX.Y.Z-N
docker inspect tonistiigi/binfmt:qemu-vX.Y.Z-N | grep "Id"
```

### Update Workflow

Update the version and digest in build.yml:

```yaml
docker run --privileged --rm \
  tonistiigi/binfmt:qemu-vX.Y.Z-N@sha256:NEW_DIGEST \
  --install arm64,amd64,arm,386,ppc64le,s390x,mips64le
```

## Updating GitHub Actions

**Files**:
- [.github/workflows/build.yml](.github/workflows/build.yml)
- [.github/workflows/release.yml](.github/workflows/release.yml)

### Get Commit SHA for Version

1. Visit releases: https://github.com/actions/checkout/releases
2. Click on version tag (e.g., v4.1.7)
3. Copy full commit SHA from URL or commits page

### Update Workflow

Replace:
```yaml
uses: actions/checkout@v4
```

With:
```yaml
# Pin to commit SHA for security
# actions/checkout v4.1.7
uses: actions/checkout@692973e3d937129bcbf40652eb9f2f61becf3332
```

## Security Vulnerability Response

If a security vulnerability is discovered in a pinned dependency:

1. **Assess Impact**: Determine if the vulnerability affects our builds
2. **Update Immediately**: Get fixed version and update pins
3. **Test**: Run full build matrix to ensure compatibility
4. **Deploy**: Merge update as soon as tests pass
5. **Document**: Note security update in commit message

## Version Update Checklist

When updating dependency versions:

- [ ] Update version numbers/digests in relevant files
- [ ] Update "Last updated" comments
- [ ] Test builds locally or in CI
- [ ] Verify all 25 platform builds succeed
- [ ] Check for breaking changes in dependency release notes
- [ ] Update this documentation if process changes
- [ ] Commit with clear description of what was updated

## Reproducibility Verification

To verify builds are reproducible:

```bash
# Build twice with same parameters
docker build -f dockerfiles/Dockerfile.linux-musl --build-arg POSTGRESQL_VERSION=18.0.0 -t pg-test1 .
docker build -f dockerfiles/Dockerfile.linux-musl --build-arg POSTGRESQL_VERSION=18.0.0 -t pg-test2 .

# Extract and compare binaries
docker create --name pg1 pg-test1
docker create --name pg2 pg-test2
docker cp pg1:/opt/postgresql ./postgresql1
docker cp pg2:/opt/postgresql ./postgresql2
shasum -a 256 ./postgresql1/bin/postgres ./postgresql2/bin/postgres
# Checksums should match
```

## Notes

- Alpine and Debian package versions are stable within their respective releases
- Homebrew packages can change more frequently - requires more vigilance
- Docker image digests are immutable - safe to pin long-term
- GitHub Actions commit SHAs are immutable - safe to pin long-term
- Always test in non-production environment first
- Keep audit trail of all dependency updates

## Questions?

For questions about dependency updates, refer to:
- [SECURITY.md](SECURITY.md) - Security policies
- [README.md](../README.md) - Project overview
- [Implementation Plan](thoughts/2025-10-21-PLAN-security-hardening-and-release-improvements.md) - Overall security strategy
