---
date: 2025-10-21T12:26:49-04:00
researcher: Claude Code
git_commit: 030bfa8ad9ec4283c793882a533b8a2a874f8dab
branch: master
repository: postgresql-binaries
topic: "Security Threat Analysis and Release Pipeline Documentation"
tags: [research, security, supply-chain, release-pipeline, docker, github-actions, attestations, slsa]
status: complete
last_updated: 2025-10-21
last_updated_by: Claude Code
last_updated_note: "Added Section 11: GitHub Attestations & Release Process Improvements"
---

# Research: Security Threat Analysis and Release Pipeline Documentation

**Date**: 2025-10-21T12:26:49-04:00
**Researcher**: Claude Code
**Git Commit**: 030bfa8ad9ec4283c793882a533b8a2a874f8dab
**Branch**: master
**Repository**: postgresql-binaries

## Research Question

Analyze the PostgreSQL binaries codebase (cloned from public source) for:
1. Potential security threats across all components
2. Detailed explanation of how the release pipeline works

## Executive Summary

This codebase builds and distributes PostgreSQL binaries for 25 different platform configurations. The analysis reveals **3 critical security vulnerabilities**, **5 high-risk issues**, and **numerous medium/low-risk concerns** across Docker builds, GitHub Actions workflows, shell scripts, and supply chain dependencies.

### Critical Security Posture

**CRITICAL RISKS:**
- SSL certificate verification is **intentionally disabled** when downloading PostgreSQL source code
- No cryptographic verification of source code authenticity (no GPG signatures, no commit hash pinning)
- Windows binaries downloaded from third-party (EnterpriseDB) without any integrity verification

**SUPPLY CHAIN EXPOSURE:**
- 30+ build dependencies without version pinning
- Privileged Docker containers running unpinned third-party images
- No Software Bill of Materials (SBOM) generation
- Build reproducibility cannot be guaranteed

**RELEASE PIPELINE:**
- Manual trigger with local validation checks
- 25 parallel platform builds on GitHub Actions
- Automated publication after successful builds (~60 min total)
- No manual approval gate before public release

---

## 1. Critical Security Vulnerabilities

### 1.1 SSL Verification Disabled During Source Code Download

**Severity**: CRITICAL
**Impact**: Man-in-the-Middle (MITM) attacks, malicious code injection

**Location**:
- [dockerfiles/Dockerfile.linux-musl:46-51](dockerfiles/Dockerfile.linux-musl)
- [dockerfiles/Dockerfile.linux-gnu:41-46](dockerfiles/Dockerfile.linux-gnu)

**Code**:
```dockerfile
RUN branch=$(echo "$POSTGRESQL_VERSION" | awk -F. '{print "REL_"$1"_"$2}') && \
    echo "branch=$branch" && \
    git config --global http.version HTTP/1.1 && \
    git config --global http.sslVerify false && \  # ← CRITICAL ISSUE
    for i in $(seq 1 5); do \
        git clone --depth 1 --branch $branch -c advice.detachedHead=false https://git.postgresql.org/git/postgresql.git /usr/src/postgresql \
            && break || sleep 3; \
    done
```

**Description**:
Both Docker builds explicitly disable SSL certificate verification with `git config --global http.sslVerify false`. This completely undermines the security of HTTPS connections and makes the build process vulnerable to man-in-the-middle attacks.

**Attack Scenario**:
1. Attacker intercepts network traffic between GitHub Actions runner and git.postgresql.org
2. Attacker serves malicious PostgreSQL source code
3. Build proceeds with compromised source (no SSL verification to detect tampering)
4. Malicious binaries are compiled and distributed to end users

**Additional Concerns**:
- HTTP/1.1 downgrade (line 46/41) may expose to protocol-level vulnerabilities
- Retry logic (5 attempts) suggests network reliability issues but doesn't justify disabling SSL
- Also present in macOS builds ([.github/workflows/build.yml:262](.github/workflows/build.yml))

---

### 1.2 No Source Code Integrity Verification

**Severity**: CRITICAL
**Impact**: No guarantee of code authenticity, supply chain compromise

**Location**:
- [dockerfiles/Dockerfile.linux-musl:44-51](dockerfiles/Dockerfile.linux-musl)
- [dockerfiles/Dockerfile.linux-gnu:39-46](dockerfiles/Dockerfile.linux-gnu)
- [.github/workflows/build.yml:257-263](.github/workflows/build.yml)

**Description**:
PostgreSQL source code is cloned without any integrity verification:
- No GPG signature verification on commits or tags
- No commit hash pinning (uses mutable branch names like `REL_18_0`)
- No checksum validation
- No verification that code is from authorized PostgreSQL maintainers

**Current Implementation**:
```bash
# Only branch names used for version selection
branch=$(echo "$VERSION" | awk -F. '{print "REL_"$1"_"$2}')
git clone --depth 1 --branch $branch ...
```

**Risk**:
- If git.postgresql.org is compromised, malicious code would be built
- Branch history could be rewritten without detection
- No forensic trail for source code provenance
- Combined with disabled SSL verification (1.1), this creates severe supply chain vulnerability

**Recommendation**:
- Pin to specific commit SHAs
- Implement GPG signature verification for tags
- Consider using official PostgreSQL release tarballs with GPG signatures

---

### 1.3 Unverified Windows Binary Downloads

**Severity**: CRITICAL
**Impact**: Complete trust in third-party binaries, no tamper detection

**Location**: [.github/workflows/build.yml:332-343](.github/workflows/build.yml)

**Code**:
```yaml
- name: Download binaries (Windows)
  if: ${{ startsWith(matrix.id, 'windows-') }}
  run: |
    postgresql_version=$(echo "$VERSION" | awk -F. '{print $1"."$2}')
    curl https://get.enterprisedb.com/postgresql/postgresql-${postgresql_version}-1-windows-x64-binaries.zip > postgresql.zip

- name: Extract binaries (Windows)
  if: ${{ startsWith(matrix.id, 'windows-') }}
  run: |
    unzip postgresql.zip
    rm -rf pgsql/doc pgsql/pgAdmin*
    mv pgsql "$INSTALL_DIRECTORY"
```

**Security Issues**:
1. **No checksum verification** - File downloaded without validating integrity
2. **No signature verification** - No GPG/PGP signature check
3. **No HTTPS certificate validation** - Basic curl without explicit cert verification
4. **Complete third-party trust** - Total reliance on EnterpriseDB infrastructure security
5. **Immediate extraction and use** - Compromised binary would be directly included in release

**Attack Scenario**:
1. Attacker compromises EnterpriseDB's CDN or performs MITM attack
2. Serves malicious PostgreSQL Windows binaries
3. Build proceeds without any validation
4. Malicious binaries packaged and distributed to Windows users

**Recommendation**:
- Obtain and verify checksums from EnterpriseDB
- Implement GPG signature verification if available
- Consider building Windows binaries from source instead of downloading

---

## 2. High-Risk Security Issues

### 2.1 Unpinned Package Dependencies (Linux Builds)

**Severity**: HIGH
**Impact**: Build reproducibility compromised, potential for dependency confusion

**Location**:
- [dockerfiles/Dockerfile.linux-musl:6-42](dockerfiles/Dockerfile.linux-musl)
- [dockerfiles/Dockerfile.linux-gnu:5-37](dockerfiles/Dockerfile.linux-gnu)

**Code Example** (MUSL):
```dockerfile
RUN apk update
RUN apk add --no-cache \
      bash \
      bison \
      clang16 \
      dpkg \
      dpkg-dev \
      gcc \
      g++ \
      git \
      openssl-dev \
      python3 \
      # ... 30+ packages WITHOUT version pinning
```

**Critical Unpinned Packages**:
- **Compilers**: `clang16`, `gcc`, `g++` - compiler versions affect binary security
- **Security Libraries**: `openssl-dev` - cryptographic library updates critical
- **Build Tools**: `git`, `make`, `bison`, `flex` - could introduce malicious build behavior
- **Runtime Dependencies**: `python3`, `perl` - could affect extension security

**Risk**:
- Different builds at different times pull different package versions
- Builds are not reproducible
- Package updates could introduce vulnerabilities
- No control over transitive dependencies
- Dependency confusion attacks possible

**Recommendation**:
- Pin all package versions (e.g., `clang16=16.0.6-r0`)
- Generate and commit lock file for Alpine/Debian packages
- Regularly audit and update pinned versions

---

### 2.2 Unpinned Homebrew Packages (macOS)

**Severity**: HIGH
**Impact**: Build reproducibility compromised, critical dependency version drift

**Location**: [.github/workflows/build.yml:268-278](.github/workflows/build.yml)

**Code**:
```yaml
brew install \
  fop \
  gettext \
  icu4c \
  lld \
  llvm \
  lz4 \
  openssl \
  readline \
  xz \
  zstd
```

**Critical Dependencies**:
- `llvm` - LLVM infrastructure for compilation
- `openssl` - Cryptographic library
- `icu4c` - Unicode/internationalization library

**Risk**:
- Homebrew always installs latest versions
- No build reproducibility across time
- Breaking changes could fail builds unexpectedly
- Security vulnerabilities in dependencies unpredictable

**Recommendation**:
- Pin Homebrew formula versions or commit SHAs
- Use Homebrew Bundle with Brewfile.lock.json

---

### 2.3 Privileged Docker Container with Unpinned Image

**Severity**: HIGH
**Impact**: Full system compromise possible if image is malicious

**Location**: [.github/workflows/build.yml:233](.github/workflows/build.yml)

**Code**:
```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

**Security Issues**:
1. **Privileged mode** - Grants unrestricted access to host kernel
2. **No version pinning** - Uses latest tag implicitly (no `:v1.2.3` or `@sha256:...`)
3. **Third-party image** - Complete trust in `tonistiigi/binfmt` maintainer
4. **No signature verification** - No Docker Content Trust
5. **Broad scope** - `--install all` registers all architecture emulators (unnecessary attack surface)

**Attack Scenario**:
1. Attacker compromises tonistiigi/binfmt image on Docker Hub
2. Malicious image pulled during build
3. Privileged container has full host access
4. Could exfiltrate secrets, modify artifacts, or compromise runner

**Recommendation**:
- Pin to specific version: `tonistiigi/binfmt:v7.0.0` or digest `@sha256:...`
- Enable Docker Content Trust
- Only install needed architectures, not `--install all`
- Consider alternative approaches that don't require privileged mode

---

### 2.4 GitHub Actions Not Pinned to Commit SHA

**Severity**: MEDIUM-HIGH
**Impact**: Supply chain attack via compromised GitHub Actions

**Location**:
- [.github/workflows/build.yml:202](.github/workflows/build.yml)
- [.github/workflows/release.yml:17,44](.github/workflows/release.yml)

**Code**:
```yaml
- name: Checkout source
  uses: actions/checkout@v4  # ← Should be commit SHA
```

**Risk**:
- `@v4` is a mutable tag that can be updated by action maintainer
- If actions/checkout repository is compromised, malicious code could run
- Recommended practice is immutable commit SHA with version comment

**Best Practice**:
```yaml
uses: actions/checkout@8e5e7e5ab8b370d6c329ec480221332ada57f0ab  # v4.1.1
```

---

### 2.5 Build Argument Injection Vulnerability

**Severity**: HIGH
**Impact**: Command injection, arbitrary code execution during build

**Location**:
- [dockerfiles/Dockerfile.linux-musl:3,44,58](dockerfiles/Dockerfile.linux-musl)
- [dockerfiles/Dockerfile.linux-gnu:3,39,53](dockerfiles/Dockerfile.linux-gnu)

**Code**:
```dockerfile
ARG POSTGRESQL_VERSION

RUN branch=$(echo "$POSTGRESQL_VERSION" | awk -F. '{print "REL_"$1"_"$2}') && \
    ...
    git clone --depth 1 --branch $branch ...

RUN major_version=$(echo "$POSTGRESQL_VERSION" | awk -F. '{print $1}') && \
    ./configure \
      $([ $major_version -le 16 ] && echo "--enable-thread-safety") \
      ...
```

**Vulnerability**:
- `POSTGRESQL_VERSION` build argument used in shell commands without validation
- While `awk` provides some sanitization, malicious input could cause unexpected behavior
- `$branch` variable not quoted in git clone command (line 49/44)

**Potential Exploit**:
```bash
docker build --build-arg POSTGRESQL_VERSION='18.0; malicious_command' .
```

**Recommendation**:
- Validate `POSTGRESQL_VERSION` format with regex before use
- Quote all variable references
- Use parameterized commands where possible

---

## 3. Supply Chain Security Analysis

### 3.1 External Dependencies Summary

**PostgreSQL Source Code**:
- **Source**: https://git.postgresql.org/git/postgresql.git
- **Verification**: ❌ None (SSL disabled, no signature verification)
- **Pinning**: ❌ Branch names only (mutable)

**Windows Binaries**:
- **Source**: https://get.enterprisedb.com/postgresql/
- **Verification**: ❌ None (no checksum, no signature)
- **Pinning**: ❌ Major.minor version only

**Docker Base Images**:
- **Alpine**: `alpine:3.19.0` - ✅ Version pinned, ❌ Not digest-pinned
- **Debian**: `debian:12.4` - ✅ Version pinned, ❌ Not digest-pinned
- **Third-party**: `tonistiigi/binfmt` - ❌ No version pinning

**System Packages**:
- **Alpine APK**: ❌ 30+ packages unpinned
- **Debian APT**: ❌ 30+ packages unpinned
- **Homebrew**: ❌ 11 packages unpinned

**GitHub Actions**:
- **actions/checkout@v4**: ❌ Mutable tag (should be commit SHA)

### 3.2 Dependency Lock Files

**Status**: ❌ None found

**Impact**:
- No package-lock.json, Cargo.lock, go.sum, or equivalent
- Cannot guarantee reproducible builds
- No record of exact dependency versions used
- Difficult to audit what dependencies were in production builds

---

### 3.3 Software Bill of Materials (SBOM)

**Status**: ❌ Not generated

**Impact**:
- No transparency about included components
- Difficult to track vulnerabilities in dependencies
- Missing modern supply chain security best practice
- No compliance with Executive Order 14028 (if applicable)

**Recommendation**:
- Generate SBOM in SPDX or CycloneDX format
- Include in release artifacts
- Automate SBOM generation in CI/CD

---

## 4. Docker Security Findings

### 4.1 SSL/TLS Security

**Finding**: SSL verification disabled (CRITICAL - covered in section 1.1)

---

### 4.2 User Privilege Management

**Finding**: ✅ Good practice - non-root user at runtime

**MUSL** ([dockerfiles/Dockerfile.linux-musl:92-97](dockerfiles/Dockerfile.linux-musl)):
```dockerfile
RUN addgroup -S -g 1000 postgresql && \
    adduser -D -S -G postgresql -u 1000 -s /bin/ash postgresql && \
    mkdir -p /opt/test && \
    chown postgresql:postgresql /opt/test

USER postgresql
```

**GNU** ([dockerfiles/Dockerfile.linux-gnu:85-90](dockerfiles/Dockerfile.linux-gnu)):
```dockerfile
RUN groupadd --system --gid 1000 postgresql && \
    useradd --system --gid postgresql --uid 1000 --shell /bin/bash --create-home postgresql && \
    mkdir -p /opt/test && \
    chown postgresql:postgresql /opt/test

USER postgresql
```

**Assessment**: Proper privilege demotion - builds run as root but final user is non-privileged.

---

### 4.3 RPATH Security Concerns

**Severity**: MEDIUM
**Impact**: Potential for local privilege escalation via library injection

**Location**:
- [dockerfiles/Dockerfile.linux-musl:83-87](dockerfiles/Dockerfile.linux-musl)
- [dockerfiles/Dockerfile.linux-gnu:76-80](dockerfiles/Dockerfile.linux-gnu)

**Code** (MUSL):
```dockerfile
# Update binary rpath to use a relative path.
RUN cd /opt/postgresql/bin && \
    rpath=$(patchelf --print-rpath postgres | sed 's|/opt/postgresql/lib|$ORIGIN/../lib|g') && \
    find ./ -type f -print0 | while IFS= read -r -d '' file; do patchelf --set-rpath $rpath $file; done
```

**Security Analysis**:

**Positive**:
- Enables binary relocation (portability)
- Uses `$ORIGIN` relative paths instead of absolute paths
- Prevents `LD_LIBRARY_PATH` dependency (reduces injection risk)

**Concerns**:
- `$ORIGIN/../lib` allows loading libraries from relative paths
- If attacker can write to `../lib` relative to binary location, could inject malicious libraries
- Applies to ALL files in bin/ directory without type checking
- GNU version uses less-safe `find ... | xargs` (potential issues with special characters)

**Similar Issue - macOS**: [.github/workflows/build.yml:323-326](.github/workflows/build.yml)
```bash
find $INSTALL_DIRECTORY/bin -type f | xargs -L 1 install_name_tool -change $INSTALL_DIRECTORY/lib/libpq.5.dylib '@executable_path/../lib/libpq.5.dylib'
```

---

### 4.4 Image Layer Caching

**Severity**: MEDIUM
**Impact**: Compromised layers could persist across builds

**Issue**: Git clone with disabled SSL may be cached in Docker layers. If a layer contains compromised source code, subsequent builds might reuse it without re-validation.

**Recommendation**: Use `--no-cache` for security-sensitive build steps or implement proper source verification.

---

## 5. GitHub Actions Workflow Security

### 5.1 Permissions Model

**release.yml** ([.github/workflows/release.yml:8-9](.github/workflows/release.yml)):
```yaml
permissions:
  contents: write  # ← Broad permission
```

**ci.yml** ([.github/workflows/ci.yml:11-12](.github/workflows/ci.yml)):
```yaml
permissions:
  contents: read  # ← Good practice
```

**build.yml**: ❌ No explicit permissions (inherits from caller)

**Issue**: Build workflow inherits `contents: write` when called from release.yml. If build is compromised, attacker has repository write access.

**Recommendation**: Define explicit minimal permissions in build.yml.

---

### 5.2 Secret Exposure Risk

**Location**: Multiple workflow steps

**Code** ([.github/workflows/build.yml:437-442](.github/workflows/build.yml)):
```yaml
- name: Upload release archive
  if: ${{ inputs.release == true }}
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # ← Token in environment
  run: |
    gh release upload "$VERSION" ${{ env.ASSET }} ${{ env.ASSET_SUM }}
```

**Risk**:
- `GITHUB_TOKEN` passed as environment variable to shell script
- If `$VERSION`, `$ASSET`, or `$ASSET_SUM` contain malicious content, token could be exposed
- Command injection could exfiltrate token

**Recommendation**: Use GitHub CLI with explicit token passing via stdin or file.

---

### 5.3 Environment Variable Injection

**Severity**: HIGH
**Location**: [.github/workflows/build.yml:208-222](.github/workflows/build.yml)

**Code**:
```bash
version=$(echo "${{ github.ref_name }}" | grep '^[0-9]*.[0-9]*.[0-9]*$') || true

if [ -z "$version" ]; then
  version="18.0.0"
fi

archive="postgresql-$version-${{ matrix.target }}"
install_directory="$root_directory/$archive"

echo "ARCHIVE=$archive" | tee -a $GITHUB_ENV
echo "INSTALL_DIRECTORY=$install_directory" | tee -a $GITHUB_ENV
echo "VERSION=$version" | tee -a $GITHUB_ENV
```

**Vulnerability**:
- `github.ref_name` is user-controllable via tag/branch names
- Validation uses `|| true` which means failure is silently ignored
- Could create tags with shell metacharacters that bypass validation
- Variables written to `$GITHUB_ENV` persist across steps

**Attack Scenario**:
```bash
git tag $'18.0.0\nMALICIOUS_VAR=evil'
git push origin $'18.0.0\nMALICIOUS_VAR=evil'
```

**Recommendation**:
- Remove `|| true` - fail on validation failure
- Sanitize input before use
- Use stronger validation regex

---

### 5.4 Pull Request Trigger Without Explicit Approval

**Severity**: MEDIUM
**Location**: [.github/workflows/ci.yml:7-9](.github/workflows/ci.yml)

**Code**:
```yaml
pull_request:
  branches:
    - main
```

**Issue**: Workflow triggers on any PR to main, including from forks.

**Risk**: Malicious actors could submit PRs executing arbitrary code in workflow context.

**Note**: GitHub provides some protections for first-time contributors, but workflow doesn't document these assumptions.

**Recommendation**: Add comment documenting reliance on GitHub's fork PR protections or require explicit approval.

---

### 5.5 Tag-Based Release Trigger

**Severity**: HIGH
**Location**: [.github/workflows/release.yml:4-6](.github/workflows/release.yml)

**Code**:
```yaml
on:
  push:
    tags:
      - "[0-9]+.[0-9]+.[0-9]+"
```

**Issue**: Any push of matching tag triggers release with write permissions.

**Risk**: If repository write access is compromised, attacker could push malicious tags to trigger releases with altered binaries.

**Recommendation**:
- Implement tag signing verification
- Require protected tags (GitHub Enterprise feature)
- Add manual approval step before publication

---

## 6. Shell Script Security

### 6.1 test.sh Security Issues

**Location**: [scripts/test.sh](scripts/test.sh)

**High-Risk Findings**:

1. **Unvalidated Command-Line Argument** (Line 5):
   ```bash
   postgresql_version=$(echo "$1" | awk -F. '{print ""$1"."$2}')
   ```
   - No validation of `$1`
   - Continues with empty `postgresql_version` if input missing

2. **Path Traversal Risk** (Lines 7, 12):
   ```bash
   test_directory="$(pwd)"
   cd "$test_directory/bin"
   ```
   - Assumes current directory is safe
   - Could execute binaries from untrusted paths

3. **Relative Path Execution** (Lines 13-23):
   ```bash
   ./initdb ...
   ./pg_ctl ...
   ./psql ...
   ```
   - Executes binaries from relative paths without verification
   - If attacker places malicious binaries in current directory, they'll be executed

4. **No Environment Sanitization** (Throughout):
   - Doesn't control `PATH`, `LD_PRELOAD`, `LD_LIBRARY_PATH`
   - Malicious environment variables could redirect operations

5. **Trust Authentication** (Line 13):
   ```bash
   ./initdb -A trust -U postgres -D "$data_directory" -E UTF8
   ```
   - Disables authentication completely
   - Acceptable for testing but risky if database is network-accessible

---

### 6.2 release.sh Security Issues

**Location**: [scripts/release.sh](scripts/release.sh)

**Critical Findings**:

1. **No PATH Sanitization** (Throughout):
   - Relies on `git` in PATH without verification
   - Attacker controlling PATH could substitute malicious git binary

2. **No GPG Signing** (Line 68):
   ```bash
   git tag "$version" --file="${release_notes}" --cleanup=strip
   ```
   - Tags not signed with `-s` flag
   - Tag authenticity unverifiable

3. **Remote Origin Not Validated** (Line 69):
   ```bash
   git push origin "$version"
   ```
   - Doesn't verify `origin` points to expected repository
   - Attacker could modify git config to redirect pushes

4. **Batch Tag Creation** (Lines 66-70):
   - If error occurs mid-loop, some tags pushed and some aren't
   - Leaves repository in inconsistent state
   - No rollback mechanism

5. **No Release Notes Validation** (Lines 49-52):
   - Content of `release_notes.md` not validated
   - Malicious markdown or special characters could be injected into tags

6. **No Authentication Verification** (Before line 69):
   - Doesn't verify user has authentication credentials
   - Tags could be created locally but fail to push

---

## 7. Complete Release Pipeline Documentation

### 7.1 Release Initiation (Manual)

**Trigger**: Developer runs [scripts/release.sh](scripts/release.sh) locally

**Prerequisites**:
- Must be on `main` branch
- Working directory must be clean (no uncommitted changes)
- Local branch synchronized with `origin/main`
- `release_notes.md` file must exist

**Process** (scripts/release.sh):

1. **Version Initialization** (Lines 5-21):
   ```bash
   release="0"
   all_versions=( "18.0" "17.6" "17.5" ... "13.0" )
   # Adds .0 suffix to each version
   # Sorts versions numerically
   ```

2. **Pre-Flight Validations** (Lines 23-52):
   - Branch must be `main` (lines 24-27)
   - No uncommitted changes (lines 30-33)
   - Branch up-to-date with origin (lines 36-39)
   - Versions don't already exist as tags (lines 42-47)
   - Release notes file exists (lines 49-52)

3. **User Confirmation** (Lines 54-64):
   ```bash
   echo "You are about to release the following versions:"
   echo "${versions[@]}" | tr ' ' ',' | sed 's/,/, /g'
   read -r -p "Do you want to proceed? (y/N): " userInput
   ```
   - **ONLY manual approval gate in entire pipeline**

4. **Tag Creation and Push** (Lines 66-70):
   ```bash
   for version in "${versions[@]}"; do
     echo "Creating version ${version}"
     git tag "$version" --file="${release_notes}" --cleanup=strip
     git push origin "$version"  # ← Triggers GitHub workflow
   done
   ```

**Example**: Creating tags `13.0.0`, `13.1.0`, ... `18.0.0` (70+ versions per release)

---

### 7.2 GitHub Workflow Orchestration

**Workflow Trigger**: [.github/workflows/release.yml:4-6](.github/workflows/release.yml)

```yaml
on:
  push:
    tags:
      - "[0-9]+.[0-9]+.[0-9]+"  # Matches any semantic version tag
```

**Job Flow**:
```
create_release (release.yml)
    ↓
build (release.yml) → calls build.yml
    ↓    ↓    ↓    ↓
  [25 parallel matrix builds]
    ↓    ↓    ↓    ↓
publish_release (release.yml)
```

---

### 7.3 Job 1: Create Release (Draft)

**Workflow**: [.github/workflows/release.yml:12-29](.github/workflows/release.yml)

**Runner**: ubuntu-latest

**Steps**:
1. Checkout source code (line 17)
2. Extract version from tag name (lines 18-23):
   ```bash
   echo "VERSION=${{ github.ref_name }}" | tee -a $GITHUB_ENV
   ```
3. Create draft GitHub release (lines 24-27):
   ```bash
   gh release create "${VERSION}" --draft --verify-tag --title $VERSION
   ```

**Key**: `--draft` flag means release is invisible to public

**Output**: `version` (line 29) - passed to build job

---

### 7.4 Job 2: Build All Platforms (25 Parallel Jobs)

**Workflow**: [.github/workflows/build.yml](.github/workflows/build.yml)
**Called from**: [.github/workflows/release.yml:31-36](.github/workflows/release.yml)

**Input**: `release: true` (triggers artifact upload)

**Platform Matrix** (Lines 21-44, 53-195):

**Linux** (21 targets):
- ARM: arm64, arm64-musl, arm, arm-musl, armhf, armhf-musl, armv5te, armv7l, armv7l-musl
- x86: x64, x64-musl, x86, x86-musl, i586, i586-musl
- Other: mips64le, powerpc64el, powerpc64el-musl, s390x, s390x-musl

**macOS** (2 targets):
- arm64 (Apple Silicon) - runs on macos-15
- x64 (Intel) - runs on macos-13

**Windows** (1 target):
- x64 (MSVC)

**Matrix Strategy**: `fail-fast: false` - all builds continue even if one fails

---

#### 7.4.1 Linux Build Process

**Environment Setup** (Lines 206-222):
```bash
version=$(echo "${{ github.ref_name }}" | grep '^[0-9]*.[0-9]*.[0-9]*$') || true
if [ -z "$version" ]; then
  version="18.0.0"  # Default for non-release builds
fi
```

**Emulation Setup** (Lines 228-241):
```bash
# Install QEMU emulators for cross-compilation
docker run --privileged --rm tonistiigi/binfmt --install all

# Select Dockerfile based on libc type
if [[ "${{ matrix.id }}" = *musl* ]]; then
  echo "DOCKERFILE=dockerfiles/Dockerfile.linux-musl" | tee -a $GITHUB_ENV
else
  echo "DOCKERFILE=dockerfiles/Dockerfile.linux-gnu" | tee -a $GITHUB_ENV
fi
```

**Docker Build** (Lines 243-251):
```bash
cp $DOCKERFILE Dockerfile
docker buildx build \
  --build-arg "POSTGRESQL_VERSION=$VERSION" \
  --platform "$PLATFORM" \
  --tag postgresql-build:latest .

# Extract compiled binaries from container
docker create --platform "$PLATFORM" --name pgbuild postgresql-build:latest
docker cp pgbuild:/opt/postgresql $INSTALL_DIRECTORY
docker rm -f pgbuild
```

**Dockerfile Execution** (Inside Docker):

1. **Source Download** (Dockerfile.linux-musl:44-51):
   ```dockerfile
   # Clone PostgreSQL source from official repo
   branch=$(echo "$POSTGRESQL_VERSION" | awk -F. '{print "REL_"$1"_"$2}')
   git clone --depth 1 --branch $branch https://git.postgresql.org/git/postgresql.git
   ```

2. **Compilation** (Dockerfile.linux-musl:58-81):
   ```dockerfile
   ./configure --prefix /opt/postgresql --with-openssl --with-gssapi ...
   make world-bin
   make install-world-bin
   make -C contrib install
   ```

3. **RPATH Adjustment** (Dockerfile.linux-musl:83-87):
   ```dockerfile
   # Make binaries relocatable
   patchelf --set-rpath '$ORIGIN/../lib' /opt/postgresql/bin/*
   ```

4. **User Setup** (Dockerfile.linux-musl:92-97):
   ```dockerfile
   adduser postgresql (UID 1000)
   USER postgresql
   ```

---

#### 7.4.2 macOS Build Process

**Source Checkout** (Lines 257-263):
```bash
branch=$(echo "$VERSION" | awk -F. '{print "REL_"$1"_"$2}')
git clone --depth 1 --branch $branch https://git.postgresql.org/git/postgresql.git "$source_directory"
```

**Dependency Installation** (Lines 268-278):
```bash
brew install fop gettext icu4c lld llvm lz4 openssl readline xz zstd
```

**Compilation** (Lines 291-320):
```bash
./configure --prefix "$INSTALL_DIRECTORY" --with-openssl --with-llvm ...
make world-bin
make install-world-bin
make -C contrib install
```

**Library Path Update** (Lines 323-326):
```bash
# Make binaries relocatable
find $INSTALL_DIRECTORY/bin -type f | \
  xargs -L 1 install_name_tool -change \
  $INSTALL_DIRECTORY/lib/libpq.5.dylib \
  '@executable_path/../lib/libpq.5.dylib'
```

---

#### 7.4.3 Windows Build Process

**Binary Download** (Lines 332-336):
```bash
postgresql_version=$(echo "$VERSION" | awk -F. '{print $1"."$2}')
curl https://get.enterprisedb.com/postgresql/postgresql-${postgresql_version}-1-windows-x64-binaries.zip > postgresql.zip
```

**Extraction and Cleanup** (Lines 338-343):
```bash
unzip postgresql.zip
rm -rf pgsql/doc pgsql/pgAdmin*  # Remove unnecessary components
mv pgsql "$INSTALL_DIRECTORY"
```

---

#### 7.4.4 Archive Creation (All Platforms)

**Preparation** (Lines 349-352):
```bash
cp README.md LICENSE "$INSTALL_DIRECTORY"
ls -l "$INSTALL_DIRECTORY/"
```

**Linux/macOS** (Lines 356-362):
```bash
tar czf "$ARCHIVE.tar.gz" "$ARCHIVE"
shasum -a 256 "$ARCHIVE.tar.gz" > "$ARCHIVE.tar.gz.sha256"
```

**Windows** (Lines 364-379):
```bash
# Primary .tar.gz (consistent with other platforms)
tar czf "$ARCHIVE.tar.gz" "$ARCHIVE"
certutil -hashfile "$ARCHIVE.tar.gz" SHA256 > "$ARCHIVE.tar.gz.sha256"

# Convenience .zip for Windows users
7z a "$ARCHIVE.zip" "$ARCHIVE"
certutil -hashfile "$ARCHIVE.zip" SHA256 > "$ARCHIVE.zip.sha256"
```

**Archive Naming**: `postgresql-{VERSION}-{TARGET}.tar.gz`
**Example**: `postgresql-18.0.0-x86_64-unknown-linux-gnu.tar.gz`

---

#### 7.4.5 Testing and Validation

**Architecture Test** (Lines 385-400):
```bash
postgres_file="$INSTALL_DIRECTORY/bin/postgres"

if [ "${{ matrix.os }}" == "ubuntu-latest" ]; then
  cpu_architecture=$(readelf --file-header "$postgres_file" | grep 'Machine:')
else
  cpu_architecture=$(file "$postgres_file")
fi

if [[ "$cpu_architecture" != *"${{ matrix.architecture }}"* ]]; then
  echo "ERROR: CPU architecture mismatch"
  exit 1
fi
```

**Functional Test** (Lines 402-431):

1. Extract archive to temporary directory
2. Run [scripts/test.sh](scripts/test.sh) on extracted files:
   - **macOS/Windows**: Native execution
   - **Linux**: Docker container with platform emulation

**Test Script** (scripts/test.sh):
```bash
# Initialize database
./initdb -A trust -U postgres -D "$data_directory" -E UTF8

# Start PostgreSQL server
./pg_ctl -w -D "$data_directory" -o "-p 65432 -F" start

# Run tests
test "$(./psql ... -c 'SHOW SERVER_VERSION')" = "$postgresql_version"
test "$(./psql ... -c 'SHOW SERVER_ENCODING')" = "UTF8"
test $(./psql ... -c "SELECT extname FROM pg_extension WHERE extname = 'plpgsql'") = "plpgsql"

# Cleanup (trap ensures server stops)
```

---

#### 7.4.6 Artifact Upload (Release Builds Only)

**Condition**: `if: ${{ inputs.release == true }}`

**Upload** (Lines 437-449):
```bash
# All platforms upload .tar.gz + .sha256
gh release upload "$VERSION" ${{ env.ASSET }} ${{ env.ASSET_SUM }}

# Windows additionally uploads .zip + .sha256
gh release upload "$VERSION" ${{ env.WINDOWS_ASSET }} ${{ env.WINDOWS_ASSET_SUM }}
```

**Uploads to**: Draft release created in Job 1

---

### 7.5 Job 3: Publish Release

**Workflow**: [.github/workflows/release.yml:38-56](.github/workflows/release.yml)

**Dependencies**: `needs: [build]` - runs only after ALL 25 builds succeed

**Runner**: ubuntu-latest

**Steps**:
1. Checkout source (lines 43-44)
2. Extract version from tag (lines 45-50)
3. **Publish release** (lines 51-54):
   ```bash
   gh release edit "${VERSION}" --draft=false
   ```

**Key**: Changes draft to published - **all artifacts now publicly visible**

**No Manual Approval**: This step is fully automated

---

### 7.6 Complete Timeline

**Release Duration**: Approximately 60 minutes from tag push to public release

**Timeline**:
- **T+0**: Developer runs release.sh, creates/pushes tags
- **T+1 min**: GitHub workflow triggered, draft release created
- **T+2 min**: 25 parallel builds start
- **T+2-60 min**: Platform builds execute (varies by complexity)
  - Linux: 15-30 min (Docker build + QEMU emulation)
  - macOS: 20-40 min (native compilation)
  - Windows: 5-10 min (download + repackage)
- **T+60 min**: All builds complete, artifacts uploaded
- **T+61 min**: publish_release job runs
- **T+62 min**: **Release published publicly**

**Critical Success Factor**: ALL 25 builds must succeed or release remains draft

---

### 7.7 Failure Handling

**Build Failure**:
- If ANY platform build fails, `publish_release` job doesn't run
- Draft release remains unpublished
- Developer can inspect failures, delete draft, fix issues, retry

**No Rollback Mechanism**:
- Tags already pushed to origin
- Would need manual tag deletion and recreation for retry

**Partial Upload Risk**:
- If upload fails for some platforms but succeeds for others
- Draft release would have incomplete artifacts
- No automatic cleanup or validation of complete artifact set

---

## 8. Recommendations

### 8.1 Immediate Actions (Critical)

1. **Remove SSL Verification Disable**
   - Delete `git config --global http.sslVerify false` from both Dockerfiles
   - If network issues exist, fix root cause (proxy configuration, certificate trust)
   - **Location**: Dockerfile.linux-musl:47, Dockerfile.linux-gnu:42

2. **Implement Source Code Verification**
   - Pin PostgreSQL source to specific commit SHAs
   - Implement GPG signature verification for tags
   - Consider using official release tarballs with signatures
   - **Alternative**: Use `git verify-tag` or `git verify-commit`

3. **Verify Windows Binaries**
   - Obtain checksums from EnterpriseDB
   - Implement checksum verification before extraction
   - Consider building from source instead of downloading
   - **Location**: .github/workflows/build.yml:332-336

---

### 8.2 High Priority

4. **Pin All Dependencies**
   - Alpine packages: Specify versions (e.g., `clang16=16.0.6-r0`)
   - Debian packages: Specify versions
   - Homebrew: Use Brewfile with version locking
   - Docker images: Use digest pinning (`@sha256:...`)
   - GitHub Actions: Pin to commit SHAs with version comments

5. **Secure Privileged Docker Container**
   - Pin `tonistiigi/binfmt` to specific version/digest
   - Enable Docker Content Trust
   - Consider alternatives that don't require privileged mode
   - Only install needed architectures (`--install arm64,amd64` not `all`)
   - **Location**: .github/workflows/build.yml:233

6. **Add Manual Approval Gate**
   - Require manual approval before publish_release job runs
   - Use GitHub Environments with required reviewers
   - Prevents automatic publication of potentially compromised builds

---

### 8.3 Medium Priority

7. **Generate SBOM**
   - Use SPDX or CycloneDX format
   - Include all dependencies (system packages, source code)
   - Publish as release artifact
   - Automate in CI/CD pipeline

8. **Implement Tag Signing**
   - Use GPG to sign all release tags
   - Verify signatures in GitHub Actions
   - Document signing key fingerprints

9. **Add Input Validation**
   - Validate `POSTGRESQL_VERSION` format before use
   - Remove `|| true` from validation (fail on invalid input)
   - Sanitize all user-controlled inputs

10. **Improve Test Coverage**
    - Add security-specific tests (SSL, authentication)
    - Verify binary hardening features (RELRO, PIE, stack canaries)
    - Test on actual hardware (not just emulation)

---

### 8.4 Low Priority

11. **Add LICENSE File**
    - Create LICENSE file for this wrapper project
    - Clarify relationship to PostgreSQL license

12. **Improve Error Handling**
    - Implement rollback mechanism for partial releases
    - Validate complete artifact set before publication
    - Better logging and error messages

13. **Security Documentation**
    - Document security assumptions
    - Publish threat model
    - Create SECURITY.md with vulnerability reporting process

---

## 9. Positive Security Practices

Despite the identified vulnerabilities, the codebase demonstrates several good security practices:

1. ✅ **Non-Root Runtime**: Binaries run as non-privileged user (UID 1000)
2. ✅ **Multi-Stage Testing**: Architecture validation + functional testing + archive integrity
3. ✅ **Hash Generation**: SHA-256 checksums for all release artifacts
4. ✅ **Draft Releases**: Creates draft before publishing (allows review)
5. ✅ **Minimal Permissions**: GitHub Actions uses read-only for CI
6. ✅ **Error Handling**: Scripts use `set -e` for fail-fast behavior
7. ✅ **Cleanup Handlers**: Proper trap for resource cleanup
8. ✅ **Platform Matrix**: Explicit targeting reduces misconfiguration
9. ✅ **Ephemeral Containers**: Build containers destroyed after use
10. ✅ **Version Pinning (Partial)**: Base images use specific versions

---

## 10. Conclusion

This PostgreSQL binaries build system has **critical security vulnerabilities** that must be addressed immediately:

- **Disabled SSL verification** creates MITM attack vector
- **No source code authentication** allows malicious code injection
- **Unverified third-party binaries** (Windows) distributed to users
- **Extensive unpinned dependencies** prevent reproducible builds

The release pipeline is well-architected with parallel builds and automated testing, but **lacks manual approval gates** before public publication.

**Immediate next steps**:
1. Enable SSL verification (remove `http.sslVerify false`)
2. Implement source code GPG signature verification
3. Add Windows binary checksum validation
4. Pin all dependencies to specific versions
5. Add manual approval before release publication

**Risk Assessment**:
- **Current State**: High risk of supply chain compromise
- **After Immediate Fixes**: Medium risk (dependencies still unpinned)
- **After All Recommendations**: Low risk (defense-in-depth implemented)

---

## Code References and Related Files

### Docker Configuration
- [dockerfiles/Dockerfile.linux-musl](dockerfiles/Dockerfile.linux-musl) - Alpine-based Linux builds
- [dockerfiles/Dockerfile.linux-gnu](dockerfiles/Dockerfile.linux-gnu) - Debian-based Linux builds

### GitHub Actions Workflows
- [.github/workflows/release.yml](.github/workflows/release.yml) - Release orchestration
- [.github/workflows/build.yml](.github/workflows/build.yml) - Platform builds (reusable)
- [.github/workflows/ci.yml](.github/workflows/ci.yml) - Continuous integration

### Shell Scripts
- [scripts/release.sh](scripts/release.sh) - Local release initiation script
- [scripts/test.sh](scripts/test.sh) - PostgreSQL functional testing

### Documentation
- [README.md](README.md) - Project documentation

---

## 11. GitHub Attestations & Release Process Improvements

This section analyzes two important improvements identified during follow-up discussion:
1. Adding GitHub attestations for build provenance
2. Replacing manual script-based releases with workflow_dispatch

### 11.1 GitHub Attestations - What They Are and Why They Matter

**Definition**: GitHub attestations create cryptographic proof about build artifacts using [Sigstore](https://www.sigstore.dev/). They generate SLSA provenance statements that prove:

- **What** was built (artifact SHA256 hash)
- **Where** it was built (GitHub Actions, specific repository)
- **How** it was built (exact workflow file, commit SHA, build parameters)
- **When** it was built (timestamp)
- **Who** triggered it (actor, event type)

**How They Work**:
```yaml
- name: Build archive
  run: tar czf "$ARCHIVE.tar.gz" "$ARCHIVE"

- name: Attest build provenance
  uses: actions/attest-build-provenance@v1
  with:
    subject-path: '${{ env.ARCHIVE }}.tar.gz'
```

This generates a `.jsonl` attestation file that users can verify:
```bash
gh attestation verify postgresql-18.0.0-x86_64-unknown-linux-gnu.tar.gz \
  --owner Torq-Interface
```

**What Verification Proves**:
- ✅ Binary was built by official GitHub Actions workflow
- ✅ Built from specific commit SHA on specific branch/tag
- ✅ Binary hasn't been tampered with after build
- ✅ Build happened on GitHub infrastructure (not local machine)
- ✅ Immutable audit trail in public transparency log (Rekor)

**What It Does NOT Prove**:
- ❌ Whether PostgreSQL source code was authentic (Critical Issue #2)
- ❌ Whether dependencies were secure (High Risk Issue #4)
- ❌ Whether build process itself was compromised

---

### 11.2 The "Garbage In, Garbage Out" Problem

**This is critical to understand**: Attestations prove you *correctly built* something, but don't validate if the inputs were legitimate.

**Current State**:
- SSL verification disabled → Could download malicious PostgreSQL source
- No GPG verification → Can't prove source is authentic
- Unpinned dependencies → Could pull compromised packages

**With Attestations**:
- Attestation proves: "This binary was built from commit X using workflow Y"
- But if commit X contains malicious code (because SSL was disabled), attestation proves you **correctly built a compromised binary**

**Analogy**: It's like a notary witnessing you sign a document - the notary proves *you* signed it, but doesn't validate if the document content is legitimate.

**Therefore**: Attestations are **most valuable AFTER fixing the critical vulnerabilities**.

---

### 11.3 How Attestations Help This Workflow

**1. User Verification**
Users can verify downloads are official and unmodified:
```bash
curl -LO https://github.com/Torq-Interface/postgresql-binaries/releases/download/18.0.0/postgresql-18.0.0-x86_64-unknown-linux-gnu.tar.gz

gh attestation verify postgresql-18.0.0-x86_64-unknown-linux-gnu.tar.gz \
  --owner Torq-Interface

# Output shows:
✓ Verification succeeded!
  Repository: Torq-Interface/postgresql-binaries
  Workflow: .github/workflows/release.yml
  Commit: 030bfa8ad9ec4283c793882a533b8a2a874f8dab
```

**2. Tamper Evidence**
If someone replaces binaries on the release page with modified versions, verification fails.

**3. Supply Chain Transparency**
Public Rekor log provides immutable audit trail of what was built, when, and by whom.

**4. SLSA Compliance**
Helps achieve **SLSA Build Level 2** (or Level 3 with hosted runners):
- Level 2: Build process is automated and generates provenance
- Level 3: Build runs on hosted infrastructure with non-falsifiable provenance

**5. Benefits for 25-Platform Matrix**
- Each of 25 platforms gets individual attestation
- Users can verify their specific platform came from correct build
- Audit trail shows all 25 builds with timestamps
- Traceability: which runner built which platform

---

### 11.4 Implementation Plan for Attestations

**Prerequisites**: Fix critical vulnerabilities first (Section 8.1)

**Step 1: Add Attestation to build.yml**

Add after archive creation (~line 380):
```yaml
# After "Build archive" step
- name: Attest build provenance
  if: ${{ inputs.release == true }}
  uses: actions/attest-build-provenance@v1
  with:
    subject-path: '${{ env.ARCHIVE }}.tar.gz'

# For Windows builds, also attest .zip
- name: Attest Windows zip
  if: ${{ inputs.release == true && startsWith(matrix.id, 'windows-') }}
  uses: actions/attest-build-provenance@v1
  with:
    subject-path: '${{ env.ARCHIVE }}.zip'
```

**Step 2: Add Permission to release.yml**

Update permissions block (~line 8):
```yaml
permissions:
  contents: write
  id-token: write  # Required for attestations
```

**Step 3: Document Verification in README.md**

```markdown
## Verifying Downloads

All releases include cryptographic attestations proving authenticity.

**To verify a download:**

1. Install GitHub CLI: https://cli.github.com/
2. Download the binary and verify:
   ```bash
   gh attestation verify postgresql-18.0.0-x86_64-unknown-linux-gnu.tar.gz \
     --owner Torq-Interface
   ```

**This proves:**
- Binary was built by our official GitHub Actions workflow
- Binary has not been tampered with after build
- Binary is traceable to exact source code commit
- Build process is auditable via public transparency log
```

**Implementation Notes**:
- Attestations are small (~few KB each)
- 25 attestations per release version
- ~1-2 seconds added per platform build
- Stored in GitHub's artifact attestation service
- No significant storage or performance impact

---

### 11.5 Current Release Process Problems

**Current Approach**: Manual [scripts/release.sh](scripts/release.sh) that:
1. Validates 70+ versions (13.0.0 through 18.0.0)
2. Creates 70+ annotated tags locally
3. Pushes each tag to origin
4. Each tag push triggers separate workflow run

**Critical Problems**:

**1. Excessive Workflow Runs**
- 70 tags × 1 workflow each = 70 separate workflow runs
- 70 workflows × 25 platform builds = 1,750 concurrent builds
- Massive GitHub Actions minutes consumption
- Difficult to monitor (70 separate workflow pages)
- No coordination between versions

**2. Local Machine Dependency**
- Requires developer's local git configuration
- Requires push access to main repository
- Needs stable network for 70+ tag pushes
- Security risk: local machine could be compromised
- No 2FA enforcement for git push operations
- Script could be modified locally before execution

**3. No Audit Trail in GitHub**
- Manual approval (user confirmation) is invisible to GitHub
- No record of who triggered release in GitHub UI
- Cannot review inputs before execution
- Difficult to satisfy compliance/audit requirements

**4. Race Conditions & Failure Handling**
- If network fails mid-push, some tags pushed, some not
- No atomic "all or nothing" behavior
- Recovery requires manual intervention
- No rollback mechanism for partial failures

**5. Release Notes Issues**
- Requires local `release_notes.md` file
- Same notes used for all 70 versions (not ideal)
- No version control of release notes
- Easy to forget to update notes between releases

**6. Scalability Issues**
- Script hardcodes version list (lines 6-13)
- Must update script to add new PostgreSQL versions
- No validation that versions match actual PostgreSQL releases

---

### 11.6 Proposed Solution: Replace Scripts with workflow_dispatch

**User Requirement**: Only need to release **1 version at a time** (not batch 70+).

**Proposal**: Replace `scripts/` folder entirely with `workflow_dispatch` triggers.

**New Approach**:

**File: `.github/workflows/release.yml`**
```yaml
name: release

on:
  workflow_dispatch:
    inputs:
      postgresql_version:
        description: 'PostgreSQL version to release (e.g., "18.0.0")'
        required: true
        type: string
      release_notes:
        description: 'Release notes (markdown)'
        required: true
        type: string
      dry_run:
        description: 'Dry run (validate without publishing)'
        required: false
        type: boolean
        default: false

permissions:
  contents: write
  id-token: write  # For attestations

jobs:
  validate_inputs:
    name: Validate Release Inputs
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.validate.outputs.version }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Validate version format
        id: validate
        run: |
          version="${{ inputs.postgresql_version }}"

          # Validate format: X.Y.Z (semantic version)
          if ! echo "$version" | grep -qE '^[0-9]+\.[0-9]+\.[0-9]+$'; then
            echo "❌ Invalid version format: $version"
            echo "Expected format: X.Y.Z (e.g., 18.0.0)"
            exit 1
          fi

          # Check if tag already exists
          if git ls-remote --tags origin | grep -q "refs/tags/$version$"; then
            echo "❌ Version $version already exists as tag"
            exit 1
          fi

          echo "✅ Version validated: $version"
          echo "version=$version" >> $GITHUB_OUTPUT

      - name: Create draft release
        if: ${{ !inputs.dry_run }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          version="${{ steps.validate.outputs.version }}"

          # Create draft release with provided notes
          gh release create "$version" \
            --draft \
            --title "PostgreSQL $version Binaries" \
            --notes "${{ inputs.release_notes }}"

  build:
    name: Build & Attest
    needs: validate_inputs
    uses: ./.github/workflows/build.yml
    with:
      ref: ${{ github.ref }}
      release: ${{ !inputs.dry_run }}
      version: ${{ needs.validate_inputs.outputs.version }}

  publish_release:
    name: Publish Release
    needs: [validate_inputs, build]
    runs-on: ubuntu-latest
    if: ${{ !inputs.dry_run }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Create and push tag
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          version="${{ needs.validate_inputs.outputs.version }}"

          # Configure git
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          # Create annotated tag
          git tag -a "$version" -m "Release PostgreSQL $version binaries"

          # Push tag
          git push origin "$version"

      - name: Publish GitHub release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          version="${{ needs.validate_inputs.outputs.version }}"

          # Change draft to published
          gh release edit "$version" --draft=false

          echo "✅ Release published: $version"
```

**Update `build.yml` to accept version parameter**:
```yaml
on:
  workflow_call:
    inputs:
      ref:
        type: string
        default: ${{ github.ref }}
      release:
        type: boolean
        default: false
      version:  # NEW: Accept version from caller
        type: string
        required: false

jobs:
  build:
    # ... existing matrix ...
    steps:
      - name: Setup environment
        run: |
          # Use provided version if available, otherwise detect from ref
          if [ -n "${{ inputs.version }}" ]; then
            version="${{ inputs.version }}"
          else
            version=$(echo "${{ github.ref_name }}" | grep '^[0-9]*.[0-9]*.[0-9]*$') || true
            if [ -z "$version" ]; then
              version="18.0.0"  # Default
            fi
          fi

          echo "VERSION=$version" | tee -a $GITHUB_ENV
          # ... rest of setup ...
```

---

### 11.7 Benefits of workflow_dispatch Approach

**1. GitHub UI/CLI Triggering**

From GitHub UI:
```
Actions → Release → Run workflow
  → Enter version: "18.0.0"
  → Enter release notes: "PostgreSQL 18.0 binaries..."
  → Dry run: false
  → Run workflow
```

From GitHub CLI:
```bash
gh workflow run release.yml \
  --field postgresql_version="18.0.0" \
  --field release_notes="PostgreSQL 18.0 binaries for all platforms" \
  --field dry_run=false
```

**2. Single Coordinated Release**
- One workflow run (not 70 separate runs)
- All 25 platforms built together
- Single monitoring page
- Easier to cancel/restart if needed

**3. Built-in Audit Trail**
- GitHub records who triggered workflow
- Input parameters logged in workflow run
- Full transparency in Actions UI
- Satisfies compliance/audit requirements
- 2FA enforcement automatic

**4. No Local Dependencies**
- Runs entirely on GitHub infrastructure
- No local git configuration required
- No network stability concerns
- Consistent execution environment
- No risk of local machine compromise

**5. Dry Run Capability**
- Test release without publishing
- Validates all 25 builds succeed
- Preview artifacts before publication
- Safe experimentation
- Catch issues before release

**6. Better Security**
- GitHub's authentication enforcement applies (2FA/SAML)
- Can use GitHub Environments with required reviewers
- No local script modification risk
- Centralized access control
- Token scoping automatic

**7. Input Validation**
- Version format validated before execution
- Duplicate version check built-in
- Release notes required (can't forget)
- Structured, visible inputs
- Type checking for inputs

**8. Environment Protection (Optional)**
- Require approval from specific users
- Time delay before execution
- Deployment branches restriction
- Secret protection policies
- Branch protection rules

**9. Better Error Handling**
- Single workflow failure = clean rollback
- No partial tag pushes
- Automatic retry mechanisms
- Structured error reporting
- Failed builds don't create tags

---

### 11.8 Migration Plan: Scripts to workflow_dispatch

**Phase 1: Fix Critical Vulnerabilities (FIRST PRIORITY)**
```
Timeline: Immediate
Priority: CRITICAL
```

1. Remove SSL verification disable from Dockerfiles
2. Implement PostgreSQL source GPG verification
3. Add Windows binary checksum validation
4. Pin all dependencies (Alpine/Debian/Homebrew)
5. Pin Docker images to digest (`@sha256:...`)
6. Pin GitHub Actions to commit SHAs

**Phase 2: Add Attestations (SECOND PRIORITY)**
```
Timeline: After Phase 1 complete
Priority: HIGH
```

7. Add `actions/attest-build-provenance@v1` to build.yml
8. Add `id-token: write` permission to release.yml
9. Update README.md with verification instructions
10. Test attestations on non-production release
11. Validate users can verify downloaded binaries

**Phase 3: Replace Scripts with workflow_dispatch (THIRD PRIORITY)**
```
Timeline: After Phase 2 complete
Priority: MEDIUM
```

12. Implement workflow_dispatch in release.yml (as shown in 11.6)
13. Update build.yml to accept version parameter
14. Add input validation logic
15. Test workflow_dispatch with dry_run=true
16. Document new release process in README
17. **Delete scripts/ folder entirely**
18. Update CONTRIBUTING.md (if exists) with new process

**Phase 4: Optional Enhancements (FUTURE)**
```
Timeline: After Phase 3 complete
Priority: LOW
```

19. Add GitHub Environment with required reviewers
20. Implement SBOM generation (Syft/CycloneDX)
21. Add security scanning (Trivy/Grype)
22. Implement tag signing verification
23. Add more comprehensive testing

---

### 11.9 Comparison: Before vs After

| Aspect | Current (Scripts) | Proposed (workflow_dispatch) |
|--------|------------------|------------------------------|
| **Trigger** | Local script execution | GitHub UI/CLI |
| **Versions per release** | 70+ batch | 1 single version |
| **Workflow runs** | 70 separate | 1 coordinated |
| **Audit trail** | None in GitHub | Full GitHub audit log |
| **Approval** | Local Y/N prompt | Optional GitHub Environment |
| **2FA enforcement** | No | Yes (GitHub auth) |
| **Dry run** | No | Yes (built-in) |
| **Input validation** | Basic shell script | Type-checked GitHub inputs |
| **Release notes** | Local file | Workflow input (required) |
| **Rollback** | Manual tag deletion | Automatic (failed builds don't tag) |
| **Local dependencies** | Git config, network | None |
| **Security risk** | Local machine compromise | GitHub infrastructure only |
| **Monitoring** | 70 separate pages | Single workflow page |
| **Attestations** | Not compatible | Fully integrated |

---

### 11.10 Implementation Checklist

**Critical Vulnerabilities (Phase 1)**:
- [ ] Remove `git config --global http.sslVerify false` from Dockerfile.linux-musl:47
- [ ] Remove `git config --global http.sslVerify false` from Dockerfile.linux-gnu:42
- [ ] Add GPG verification for PostgreSQL git tags
- [ ] Pin PostgreSQL source to specific commit SHAs
- [ ] Add checksum verification for Windows binaries (build.yml:336)
- [ ] Pin Alpine packages with versions
- [ ] Pin Debian packages with versions
- [ ] Create Brewfile for macOS dependency pinning
- [ ] Pin tonistiigi/binfmt to specific digest (build.yml:233)
- [ ] Pin actions/checkout to commit SHA (all workflows)
- [ ] Test all builds succeed with pinned dependencies

**Attestations (Phase 2)**:
- [ ] Add `actions/attest-build-provenance@v1` after archive creation (build.yml:~380)
- [ ] Add attestation for Windows .zip files
- [ ] Add `id-token: write` permission to release.yml
- [ ] Update README.md with verification instructions
- [ ] Test attestation on non-production release
- [ ] Validate attestation verification works for users
- [ ] Document attestation in SECURITY.md (if created)

**workflow_dispatch Refactor (Phase 3)**:
- [ ] Add workflow_dispatch trigger to release.yml
- [ ] Implement input validation (version format, duplicate check)
- [ ] Add version parameter to build.yml
- [ ] Update build.yml environment setup to use version input
- [ ] Implement draft release creation in validate_inputs job
- [ ] Implement tag creation/push in publish_release job
- [ ] Add dry_run support throughout workflow
- [ ] Test dry_run releases (no publication)
- [ ] Test real release for single version
- [ ] Update README.md with new release instructions
- [ ] **Delete scripts/release.sh**
- [ ] **Delete scripts/test.sh** (move logic to workflow if needed)
- [ ] Update documentation referencing old scripts
- [ ] Remove scripts/ folder from repository

**Optional Enhancements (Phase 4)**:
- [ ] Set up GitHub Environment for releases
- [ ] Add required reviewers to environment
- [ ] Implement SBOM generation
- [ ] Add security scanning (Trivy/Grype)
- [ ] Add tag signing and verification
- [ ] Create SECURITY.md with vulnerability reporting

---

### 11.11 Updated Recommendations

**IMMEDIATE (Do First)**:
1. Fix Critical Vulnerabilities (Section 8.1) - **BLOCKS EVERYTHING ELSE**
   - SSL verification
   - Source code authentication
   - Windows binary verification
   - Dependency pinning

**HIGH PRIORITY (Do Second)**:
2. Add Attestations (Section 11.4) - **After vulnerabilities fixed**
   - Build provenance for all 25 platforms
   - User verification capability
   - SLSA Level 2 compliance

**MEDIUM PRIORITY (Do Third)**:
3. Replace Scripts with workflow_dispatch (Section 11.8) - **After attestations working**
   - Single version releases
   - GitHub UI/CLI triggering
   - Remove scripts/ folder
   - Audit trail integration

**LOW PRIORITY (Do Last)**:
4. Optional Enhancements (Section 11.9)
   - SBOM generation
   - Security scanning
   - Tag signing

**Rationale**: Attestations prove the integrity of the **build process**, but that proof is only valuable if the **inputs** (source code, dependencies) are verified first. Otherwise, you're just proving you correctly built a potentially compromised binary.

---

### 11.12 Expected Outcomes

**After Phase 1 (Critical Fixes)**:
- ✅ No more MITM vulnerability during source download
- ✅ Source code authenticity guaranteed via GPG
- ✅ Windows binaries verified before distribution
- ✅ Reproducible builds with pinned dependencies
- ✅ Supply chain attack surface significantly reduced

**After Phase 2 (Attestations)**:
- ✅ Users can cryptographically verify downloads
- ✅ Tamper detection for release artifacts
- ✅ SLSA Build Level 2 achieved
- ✅ Immutable audit trail in public log
- ✅ Compliance requirements satisfied

**After Phase 3 (workflow_dispatch)**:
- ✅ No local script dependencies
- ✅ Full GitHub audit trail for releases
- ✅ Single version releases (not 70+ batch)
- ✅ Dry run testing capability
- ✅ Better security through GitHub auth
- ✅ Simplified maintenance (no scripts to update)

---

## Open Questions

1. **Why is SSL verification disabled?** - Network configuration issue? Corporate proxy? Should be documented and fixed properly.

2. **Are Windows binaries verified offline?** - Is there a separate process for validating EnterpriseDB binaries before distribution?

3. **What is the incident response plan** if a security vulnerability is discovered in a published release?

4. **Are there compliance requirements** (e.g., SLSA levels, NIST guidelines) that should be met?

5. **Who has access to push tags?** - Access control and audit logging for release triggers.

6. **Why 70+ versions per release?** - Current script releases 70+ versions at once. Is this necessary, or can releases be per-version?

7. **What is the deprecation plan** for old PostgreSQL versions (13.x, 14.x)? How long are they supported?

8. **Are checksums/signatures available** from EnterpriseDB for Windows binaries? Can we get them from official source?
