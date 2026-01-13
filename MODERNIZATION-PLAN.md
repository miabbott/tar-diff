# tar-diff Modernization, Testing, and Fedora Packaging Plan

## Executive Summary

This document outlines a comprehensive plan to modernize the tar-diff codebase, enhance testing coverage, and package the project for Fedora Linux. The plan is divided into four major phases with specific actionable tasks, with a new focus on **OCI container image integration** to fulfill the project's primary use case.

## Current State Assessment

### Project Overview

- **Purpose**: Go library and CLI tools for diffing and patching tar files
- **Current Version**: v0.1.2
- **License**: Apache 2.0
- **Primary Use Case**: Efficient OCI image layer distribution
- **Code Size**: ~1,625 lines of Go code

### Current Technology Stack

- **Go Version**: 1.14 (released February 2020, EOL) → **Target: 1.25+**
- **CI/CD**: Cirrus CI using Fedora 32 (EOL) → **Target: GitHub Actions**
- **Test Management**: None → **Target: tmt + Testing Farm**
- **Build Tools**: Make, golangci-lint v1.25.0 (outdated)
- **Dependencies**:
  - github.com/containers/image/v5 v5.4.3 (2020)
  - github.com/klauspost/compress v1.10.4 (2020)

### Code Organization

```text
pkg/common/           # Constants and version (14 lines)
pkg/tar-diff/         # Core diff generation (1,436 lines)
  ├── diff.go         # Main entry point
  ├── analysis.go     # Tar file analysis
  ├── bsdiff.go       # Binary diff algorithm
  ├── delta.go        # Delta compression
  ├── rollsum.go      # Rolling checksum
  └── stealerreader.go # Reader wrapper
pkg/tar-patch/        # Core patch application (175 lines)
  └── apply.go        # Patch logic
cmd/tar-diff/         # CLI tool (76 lines)
cmd/tar-patch/        # CLI tool (67 lines)
```

### Known Code Issues

The codebase has several issues that need to be addressed:

1. **Deprecated APIs**: Uses `io/ioutil` package (deprecated since Go 1.16)
2. **Typos in CLI**: Usage text shows "OPION" instead of "OPTION"
3. **Unaddressed TODOs** in `pkg/tar-diff/diff.go`:
   - Handle same file multiple times in tarfile
   - Handle hardlinks properly
4. **Outdated Makefile**: Uses `GO111MODULE="on"` (unnecessary since Go 1.17)
5. **Limited OCI Integration**: Despite OCI being the primary use case, there's no direct OCI image support

### Testing Status

- **Unit Tests**: ❌ **NONE** (no *_test.go files exist)
- **Integration Tests**: ✅ Shell script (tests/test.sh, 83 lines)
- **Test Coverage**: Unknown (no unit tests to measure)
- **Benchmarks**: ❌ None

### Packaging Status

- **Fedora RPM**: ❌ No spec file
- **Container Images**: ❌ None
- **Installation**: Manual via `make install`

---

## Phase 1: Codebase Modernization

### 1.1 Go Version and Module Updates

**Priority**: HIGH
**Estimated Effort**: Medium

#### Go Version Tasks

1. **Update Go Version**
   - Update `go.mod` from Go 1.14 to Go 1.25 or newer
   - Test compilation with new Go version
   - Address any deprecated API usage
   - Leverage new Go 1.25+ features where beneficial
   - No need to maintain backward compatibility with older Go versions

2. **Update Dependencies**
   - Run `go get -u ./...` to update all dependencies
   - Update `github.com/containers/image/v5` to latest v5.x
   - Update `github.com/klauspost/compress` to latest version
   - Review and test all dependency changes
   - Audit dependencies for security vulnerabilities

3. **Dependency Management**
   - Run `go mod tidy` to clean up unused dependencies
   - Consider using `dependabot` or `renovate` for automated updates
   - Document all major dependency version requirements

#### Go Version Success Criteria

- [ ] go.mod specifies Go 1.25 or later
- [ ] All dependencies updated to latest stable versions
- [ ] Code compiles without warnings on Go 1.25+
- [ ] All existing tests pass
- [ ] Code leverages modern Go idioms and features

---

### 1.2 Code Cleanup and Bug Fixes

**Priority**: HIGH
**Estimated Effort**: Low

#### Code Cleanup Tasks

1. **Replace Deprecated APIs**
   - Replace `io/ioutil` usage with modern equivalents:
     - `ioutil.Discard` → `io.Discard`
     - `ioutil.ReadAll` → `io.ReadAll`
     - `ioutil.ReadFile` → `os.ReadFile`
     - `ioutil.WriteFile` → `os.WriteFile`
     - `ioutil.TempDir` → `os.MkdirTemp`
     - `ioutil.TempFile` → `os.CreateTemp`

2. **Fix CLI Typos**
   - Fix "OPION" → "OPTION" in `cmd/tar-diff/main.go`
   - Fix "OPION" → "OPTION" in `cmd/tar-patch/main.go`

3. **Update Makefile**
   - Remove `GO111MODULE="on"` (default since Go 1.17)
   - Replace `TRAVIS` variable with GitHub Actions equivalent
   - Update golangci-lint installation to latest version
   - Remove deprecated `GO111MODULE="off"` for tool installation

4. **Address Code TODOs**
   - Implement handling for duplicate files in tar archives
   - Implement proper hardlink support
   - Document any intentional limitations

#### Code Cleanup Success Criteria

- [ ] No deprecated `io/ioutil` usage
- [ ] All CLI typos fixed
- [ ] Makefile modernized
- [ ] TODOs addressed or documented as known limitations

---

### 1.3 Code Quality and Linting

**Priority**: HIGH
**Estimated Effort**: Medium

#### Linting Tasks

1. **Update Linting Tools**
   - Update golangci-lint from v1.25.0 to latest (v1.55+)
   - Update `.golangci.yml` configuration (create if missing)
   - Enable additional linters:
     - `gofmt`, `goimports` - Code formatting
     - `govet` - Code correctness
     - `errcheck` - Error handling
     - `staticcheck` - Static analysis
     - `gosec` - Security issues
     - `ineffassign` - Unused assignments
     - `misspell` - Spelling errors
     - `gocyclo` - Cyclomatic complexity
     - `gocritic` - Code style

2. **Address Linting Issues**
   - Run updated linters on codebase
   - Fix all critical and high-priority issues
   - Document any intentional suppressions
   - Add pre-commit hooks for linting

3. **Code Formatting**
   - Run `gofmt -s -w .` to standardize formatting
   - Run `goimports -w .` to organize imports
   - Consider using `gofumpt` for stricter formatting

#### Linting Success Criteria

- [ ] golangci-lint v1.55+ configured and passing
- [ ] Zero high-priority linting issues
- [ ] All code formatted consistently
- [ ] Makefile includes `make lint` target

---

### 1.4 CLI Improvements

**Priority**: MEDIUM
**Estimated Effort**: Medium

#### CLI Enhancement Tasks

1. **Add Progress and Verbose Output**
   - Add `-v` / `--verbose` flag for detailed operation logging
   - Show progress for large file operations
   - Display statistics after completion (files processed, compression ratio, etc.)

2. **Improve Output Options**
   - Add stdout support for `tar-diff` (currently only `tar-patch` supports `-`)
   - Add `--quiet` / `-q` flag to suppress non-error output
   - Add `--json` flag for machine-readable output

3. **Add Informational Commands**
   - Add `--info` flag to display tardiff file metadata without applying
   - Add `--list` flag to list files referenced in a tardiff
   - Add `--stats` flag to show compression statistics

4. **Improve Error Messages**
   - Provide more descriptive error messages
   - Include suggestions for common issues
   - Add `--debug` flag for troubleshooting

#### CLI Success Criteria

- [ ] Verbose mode available for all commands
- [ ] Progress indicators for large operations
- [ ] Consistent flag naming across commands
- [ ] Informational commands working

---

### 1.5 CI/CD Modernization

**Priority**: HIGH
**Estimated Effort**: Medium

#### CI/CD Tasks

1. **Remove Cirrus CI**
   - Remove `.cirrus.yml` configuration file (currently using Fedora 32, EOL)
   - Migrate all CI functionality to GitHub Actions
   - Archive/document any Cirrus-specific workflows

2. **GitHub Actions Workflows**
   - Create `.github/workflows/ci.yml` for continuous integration:
     - Build and test on Linux (Fedora 40/41, Ubuntu latest)
     - Build and test on macOS (latest)
     - Build and test on Windows (latest)
     - Use Go 1.25+ (no need for version matrix)
     - Run linting with golangci-lint
     - Generate code coverage reports
     - Upload coverage to codecov.io

   - Create `.github/workflows/release.yml` for releases:
     - Automated releases with goreleaser
     - Binary artifacts for multiple platforms (Linux, macOS, Windows)
     - Multi-architecture binaries (amd64, arm64)
     - GitHub Release notes generation
     - Container image builds (optional)

   - Create `.github/workflows/security.yml` for security:
     - Dependency vulnerability scanning with govulncheck
     - SAST (Static Application Security Testing)
     - CodeQL analysis
     - Dependabot for automated dependency updates

3. **Code Coverage Integration**
   - Set up codecov.io for coverage tracking
   - Add coverage badges to README.md
   - Set minimum coverage thresholds (target: 80%+)
   - Fail CI if coverage drops below threshold

4. **GitHub Actions Best Practices**
   - Use caching for Go modules and build artifacts
   - Use matrix strategy for multi-platform builds
   - Implement proper artifact retention policies
   - Use concurrency groups to cancel outdated runs
   - Pin action versions for reproducibility

#### CI/CD Success Criteria

- [ ] Cirrus CI configuration removed
- [ ] GitHub Actions workflows fully operational
- [ ] Multi-platform testing (Linux, macOS, Windows)
- [ ] Code coverage tracking enabled with codecov.io
- [ ] Automated security scanning active
- [ ] All CI runs complete in <10 minutes
- [ ] CI badge displayed in README.md

---

### 1.6 Documentation Updates

**Priority**: MEDIUM
**Estimated Effort**: Medium

#### Documentation Tasks

1. **Update README.md**
   - Add badges (build status, coverage, Go version, license)
   - Expand installation instructions (go install, package managers, binary releases)
   - Add complete OCI workflow examples with `skopeo` and `podman`
   - Document performance characteristics:
     - When bsdiff is used vs rollsum matching
     - The 192MB threshold and its rationale
     - Memory usage considerations
   - Add troubleshooting section
   - Add comparison with alternatives (e.g., casync, zsync)

2. **Update file-format.md**
   - Add more detailed examples
   - Include diagrams of the delta format
   - Document version compatibility

3. **Add GoDoc Comments**
   - Add package-level documentation to all packages
   - Document all exported functions, types, constants
   - Add examples in GoDoc format
   - Ensure documentation renders correctly on pkg.go.dev

4. **Create Man Pages**
   - Create `tar-diff.1` man page
   - Create `tar-patch.1` man page
   - Add `make man` target to generate man pages
   - Include man pages in installation

5. **Create Additional Documentation**
   - Add CONTRIBUTING.md with development workflow
   - Add CHANGELOG.md following Keep a Changelog format
   - Create docs/ directory with:
     - Architecture overview
     - Algorithm explanation (bsdiff, rollsum)
     - Performance tuning guide
     - API reference
     - OCI integration guide

#### Documentation Success Criteria

- [ ] README.md updated with modern badges and OCI examples
- [ ] All exported symbols documented
- [ ] Documentation visible on pkg.go.dev
- [ ] Man pages created and installable
- [ ] CONTRIBUTING.md and CHANGELOG.md present
- [ ] Complete OCI workflow documented

---

## Phase 2: OCI Container Image Integration

**Priority**: HIGH
**Estimated Effort**: High

This phase addresses the primary use case: efficient OCI container image distribution.

### 2.1 OCI Image Support Design

#### Design Considerations

1. **Image Reference Handling**
   - Support Docker-style references: `registry/image:tag`
   - Support OCI image layout directories
   - Support image digests: `image@sha256:...`

2. **Layer Diffing Strategy**
   - Diff corresponding layers by position
   - Optionally diff by content similarity (for reordered layers)
   - Handle layer additions and removals

3. **Registry Integration Options**
   - Option A: Direct registry access (requires auth handling)
   - Option B: Work with locally pulled images via containers/storage
   - Option C: Work with exported OCI layouts
   - Recommendation: Start with Option C, expand to B and A

### 2.2 OCI Integration Tasks

#### Core OCI Tasks

1. **Add OCI Layout Support**
   - Read OCI image layouts from directories
   - Parse image manifests and configs
   - Extract layer information and digests

2. **Create oci-diff Command**
   - New command: `oci-diff image1 image2 output.ocidiff`
   - Support for OCI layout directories initially
   - Generate per-layer tardiffs
   - Create manifest for the diff bundle

3. **Create oci-patch Command**
   - New command: `oci-patch input.ocidiff base-image output-image`
   - Reconstruct target image from base + diff
   - Validate reconstructed layer digests

4. **Layer Matching Algorithm**
   - Match layers by digest (exact match)
   - Match layers by position (when digests differ)
   - Optional: Match by content similarity for renamed/reordered layers

5. **Diff Bundle Format**
   - Design `.ocidiff` format (directory or archive)
   - Include manifest mapping source to target layers
   - Include per-layer tardiffs
   - Include metadata for validation

#### Advanced OCI Tasks (Future)

1. **Registry Integration**
   - Pull images directly from registries
   - Push reconstructed images to registries
   - Support for authenticated registries

2. **Streaming Support**
   - Stream layers without full download
   - Reduce memory usage for large images

3. **Multi-arch Support**
   - Handle manifest lists / OCI indexes
   - Diff corresponding architecture variants

#### OCI Success Criteria

- [ ] Can diff two OCI layout directories
- [ ] Can reconstruct target image from base + diff
- [ ] Reconstructed image has identical layer digests
- [ ] Diff bundle is significantly smaller than full image transfer
- [ ] Works with real-world images (e.g., fedora:40 → fedora:41)

---

## Phase 3: Testing Enhancement

### 3.1 Unit Test Development

**Priority**: CRITICAL
**Estimated Effort**: High

#### Current Gap

**ZERO** unit tests exist in the codebase. This is the highest priority item.

#### Unit Test Tasks

1. **pkg/common Package Tests**
   - Create `pkg/common/common_test.go`
   - Test delta operation constants
   - Test header magic bytes
   - Test version string

2. **pkg/tar-diff Package Tests**
   - Create `pkg/tar-diff/rollsum_test.go`
     - Test rolling checksum algorithm
     - Test window sliding
     - Benchmark rolling checksum performance

   - Create `pkg/tar-diff/bsdiff_test.go`
     - Test binary diff generation
     - Test patch creation
     - Test edge cases (empty files, identical files, completely different)
     - Benchmark bsdiff performance

   - Create `pkg/tar-diff/delta_test.go`
     - Test delta writer
     - Test operation encoding
     - Test varint encoding/decoding
     - Test compression

   - Create `pkg/tar-diff/analysis_test.go`
     - Test tar file analysis
     - Test file metadata extraction
     - Test content hashing

   - Create `pkg/tar-diff/diff_test.go`
     - Test main Diff() function
     - Test with various tar formats
     - Test with compressed tars (gzip, bzip2, xz, zstd)
     - Test error handling
     - Test large files (>192MB bsdiff threshold)
     - Test duplicate files in tar (addresses TODO)
     - Test hardlinks (addresses TODO)

3. **pkg/tar-patch Package Tests**
   - Create `pkg/tar-patch/apply_test.go`
     - Test Apply() function
     - Test all delta operations
     - Test FilesystemDataSource
     - Test error handling (missing files, corrupted deltas)
     - Test with symlinks, hardlinks, sparse files

4. **Integration Test Expansion**
   - Enhance `tests/test.sh`:
     - Add more edge cases
     - Test with deeply nested directories
     - Test with special characters in filenames
     - Test with very large files
     - Test failure modes

#### Testing Best Practices

- Use table-driven tests where appropriate
- Test both success and failure paths
- Include boundary conditions and edge cases
- Use `t.Parallel()` for independent tests
- Use subtests with `t.Run()` for organization
- Mock external dependencies where needed
- Use test fixtures (sample tar files)

#### Unit Test Success Criteria

- [ ] All packages have comprehensive unit tests
- [ ] Code coverage ≥ 80% overall
- [ ] Critical paths have ≥ 90% coverage
- [ ] All tests pass consistently
- [ ] Tests complete in reasonable time (<30s for unit tests)

---

### 3.2 OCI-Specific Testing

**Priority**: HIGH
**Estimated Effort**: Medium

#### OCI Test Tasks

1. **Real Image Testing**
   - Test with actual OCI image layers from popular base images
   - Test Fedora image version upgrades (e.g., fedora:40 → fedora:41)
   - Test Alpine image updates (small, simple layers)
   - Test application images (compiled binaries)

2. **Layer Characteristic Testing**
   - Test with layers containing many small files (RPM-based images)
   - Test with layers containing few large files (compiled applications)
   - Test with layers near the 192MB bsdiff threshold

3. **Compression Format Testing**
   - Test cross-format: diff gzip-compressed, reconstruct for zstd validation
   - Test with zstd-compressed layers (increasingly common)
   - Test with uncompressed layers

4. **Edge Cases**
   - Test empty layers
   - Test layers with only metadata changes
   - Test images with many layers (>10)
   - Test layer reordering scenarios

#### OCI Test Success Criteria

- [ ] Tests pass with real OCI image layers
- [ ] Cross-compression-format reconstruction works
- [ ] Large layer handling verified
- [ ] Edge cases covered

---

### 3.3 Benchmark Tests

**Priority**: MEDIUM
**Estimated Effort**: Medium

#### Benchmark Tasks

1. **Create Benchmark Suite**
   - Benchmark diff generation for various file sizes
   - Benchmark patch application
   - Benchmark rolling checksum computation
   - Benchmark compression/decompression
   - Benchmark bsdiff for different similarity levels
   - Benchmark OCI layer diffing (when implemented)

2. **Performance Baseline**
   - Document current performance characteristics
   - Create benchmark comparison against previous versions
   - Track performance regression in CI
   - Compare with alternative tools (if applicable)

3. **Optimization Opportunities**
   - Profile code to identify bottlenecks
   - Consider memory optimizations
   - Evaluate parallel processing opportunities
   - Optimize for common OCI image patterns

#### Benchmark Success Criteria

- [ ] Comprehensive benchmark suite
- [ ] Performance baseline documented
- [ ] CI tracks performance regressions

---

### 3.4 Test Management with tmt and Testing Farm

**Priority**: HIGH
**Estimated Effort**: Medium

#### tmt Overview

Integrate [tmt (Test Management Tool)](https://github.com/teemtee/tmt) and [Testing Farm](https://testing-farm.io/) for comprehensive test orchestration and execution across multiple platforms and Fedora versions.

#### tmt Tasks

1. **tmt Test Plan Creation**
   - Create `plans/` directory for tmt test plans
   - Create `plans/main.fmf` with primary test plan:

     ```yaml
     summary: Main test plan for tar-diff
     discover:
       how: fmf
     prepare:
       how: install
       package:
         - golang
         - make
         - tar
         - diffutils
         - bzip2
         - xz
         - skopeo
         - podman
     execute:
       how: tmt
     ```

   - Create individual test plans for:
     - Unit tests (`plans/unit.fmf`)
     - Integration tests (`plans/integration.fmf`)
     - OCI tests (`plans/oci.fmf`)
     - RPM package tests (`plans/rpm.fmf`)
     - Multi-architecture tests (`plans/multiarch.fmf`)

2. **FMF Metadata for Tests**
   - Create `.fmf/version` file to enable FMF
   - Add metadata to existing tests:
     - Create `tests/unit/main.fmf` for Go unit tests
     - Create `tests/integration/main.fmf` for integration tests
     - Create `tests/oci/main.fmf` for OCI integration tests
     - Tag tests appropriately (tier1, tier2, smoke, etc.)
   - Document test requirements, duration, and dependencies

3. **Testing Farm Integration**
   - Create `.github/workflows/testing-farm.yml`:
     - Trigger on pull requests and commits
     - Request tests on Testing Farm infrastructure
     - Test on multiple Fedora versions (40, 41, Rawhide)
     - Test on multiple architectures (x86_64, aarch64)
     - Report results back to GitHub PR

   - Configure Testing Farm API integration:
     - Set up Testing Farm API token in GitHub Secrets
     - Configure test environments and constraints
     - Set up result reporting and artifacts

4. **Test Fixtures and Infrastructure**
   - Create `testdata/` directory with FMF metadata
   - Generate sample tar files of various types:
     - Small (<1MB)
     - Medium (1-100MB)
     - Large (>100MB)
     - Different compression formats (gzip, bzip2, xz, zstd)
     - Edge cases (empty, single file, symlinks, hardlinks)
   - Create sample OCI layouts for testing

   - Create helper scripts in `tests/lib/`:
     - `setup.sh` - Test environment preparation
     - `helpers.sh` - Common test utilities
     - `oci-helpers.sh` - OCI-specific test utilities
     - `cleanup.sh` - Test cleanup and teardown

5. **Multi-Platform Test Matrix**
   - Test on Fedora 40, 41, Rawhide
   - Test on CentOS Stream 9, 10
   - Test on RHEL 9 (if available)
   - Test on multiple architectures (x86_64, aarch64, ppc64le, s390x)
   - Test with race detector (`-race`)
   - Test with different Go toolchain versions

6. **tmt Command Integration**
   - Add `make tmt-test` target to Makefile:

     ```makefile
     tmt-test:
         tmt run
     ```

   - Add `make tmt-lint` for plan validation
   - Document tmt usage in CONTRIBUTING.md

#### tmt Success Criteria

- [ ] tmt test plans created and validated
- [ ] FMF metadata added to all tests
- [ ] Testing Farm integration operational
- [ ] Tests run automatically on PRs via Testing Farm
- [ ] Multi-platform and multi-arch testing working
- [ ] Test results visible in GitHub PR checks
- [ ] Local tmt test execution working
- [ ] Documentation for running tests with tmt

---

### 3.5 Test Infrastructure

**Priority**: MEDIUM
**Estimated Effort**: Low

#### Test Infrastructure Tasks

1. **Local Development Testing**
   - Ensure tests can run locally without tmt
   - Maintain `make test` for quick local validation
   - Document both tmt and direct test execution

2. **Test Utilities**
   - Create helper functions for test setup/teardown
   - Create tar generation utilities in `tests/lib/`
   - Create OCI layout generation utilities
   - Create comparison and validation utilities

3. **CI Test Matrix**
   - GitHub Actions: Go 1.25+ on Linux, macOS, Windows
   - Testing Farm: Multiple Fedora versions and architectures
   - Both CI systems should run on every PR
   - Fast feedback from GitHub Actions (<5 min)
   - Comprehensive feedback from Testing Farm (<30 min)

#### Test Infrastructure Success Criteria

- [ ] Reusable test fixtures available
- [ ] Test utilities reduce duplication
- [ ] Multi-platform test coverage (GitHub Actions + Testing Farm)
- [ ] Clear documentation for both CI systems

---

## Phase 4: Fedora Packaging

### 4.1 RPM Spec File Creation

**Priority**: HIGH
**Estimated Effort**: Medium

#### RPM Spec Tasks

1. **Create tar-diff.spec**
   - Follow Fedora Go Packaging Guidelines:
     - <https://docs.fedoraproject.org/en-US/packaging-guidelines/Golang/>
   - Structure:

     ```spec
     %global goipath github.com/containers/tar-diff
     Version:        0.2.0

     %gometa

     %global common_description %{expand:
     tar-diff is a golang library and set of commandline tools to diff
     and patch tar files for efficient OCI image distribution.}

     %global golicenses LICENSE
     %global godocs README.md file-format.md CODE-OF-CONDUCT.md SECURITY.md

     Name:           tar-diff
     Release:        1%{?dist}
     Summary:        Tools to diff and patch tar files for OCI images

     License:        Apache-2.0
     URL:            %{gourl}
     Source0:        %{gosource}

     %description %{common_description}

     %gopkg

     %prep
     %goprep

     %build
     %gobuild -o %{gobuilddir}/bin/tar-diff %{goipath}/cmd/tar-diff
     %gobuild -o %{gobuilddir}/bin/tar-patch %{goipath}/cmd/tar-patch

     %install
     %gopkginstall
     install -m 0755 -vd %{buildroot}%{_bindir}
     install -m 0755 -vp %{gobuilddir}/bin/tar-* %{buildroot}%{_bindir}/
     install -m 0755 -vd %{buildroot}%{_mandir}/man1
     install -m 0644 -vp docs/man/*.1 %{buildroot}%{_mandir}/man1/

     %check
     %gocheck

     %files
     %license LICENSE
     %doc README.md file-format.md
     %{_bindir}/tar-diff
     %{_bindir}/tar-patch
     %{_mandir}/man1/tar-diff.1*
     %{_mandir}/man1/tar-patch.1*

     %gopkgfiles

     %changelog
     * Mon Jan 13 2025 Your Name <your.email@example.com> - 0.2.0-1
     - Update to 0.2.0 with OCI support and modernization
     ```

2. **Packaging Considerations**
   - Handle bundled dependencies (if needed)
   - Ensure proper file permissions
   - Include man pages
   - Handle SELinux contexts if applicable

3. **Subpackages**
   - Main package: CLI tools (tar-diff, tar-patch)
   - Devel package: Go library files for development
   - Documentation package (optional)

#### RPM Spec Success Criteria

- [ ] Valid RPM spec file created
- [ ] Spec follows Fedora guidelines
- [ ] Builds successfully with `rpmbuild`
- [ ] Installs correctly with `dnf`
- [ ] Man pages included

---

### 4.2 Build System Integration

**Priority**: HIGH
**Estimated Effort**: Low

#### Build System Tasks

1. **Update Makefile**
   - Add `make srpm` target for source RPM creation
   - Add `make rpm` target for binary RPM creation
   - Add `make man` target for man page generation
   - Ensure `make install` respects DESTDIR and PREFIX
   - Add `make dist` target for source tarball creation

2. **Build Dependencies**
   - Document build-time dependencies:
     - golang >= 1.25
     - make
     - tar
     - diffutils (for tests)
     - bzip2 (for tests)
     - xz (for tests)
     - skopeo (for OCI tests)
     - podman (for OCI tests)
     - tmt (for test management, optional)
   - Document runtime dependencies (minimal/none expected)

3. **Version Management**
   - Sync version between:
     - pkg/common/version.go
     - go.mod
     - tar-diff.spec
   - Consider using git tags for versioning
   - Automate version bumping

#### Build System Success Criteria

- [ ] `make srpm` produces valid SRPM
- [ ] `make rpm` produces valid RPM
- [ ] `make man` generates man pages
- [ ] All dependencies documented
- [ ] Version synchronized across files

---

### 4.3 Fedora Package Review Process

**Priority**: MEDIUM
**Estimated Effort**: High (includes waiting time)

#### Package Review Tasks

1. **Pre-Review Preparation**
   - Create Fedora Account System (FAS) account
   - Join fedora-packagers group
   - Read Fedora packaging guidelines thoroughly
   - Run `fedora-review` tool locally

2. **COPR Repository Setup**
   - Create COPR repository for tar-diff
   - Set up automated builds from git
   - Test installation on Fedora 40/41
   - Share COPR repo for community testing

3. **Package Review Request**
   - Upload SRPM to Fedora hosting
   - Create Bugzilla review request
   - Address reviewer feedback
   - Iterate until approved

4. **Package Import**
   - Import package to Fedora dist-git
   - Set up automated builds
   - Create update for Fedora releases
   - Submit to Bodhi for testing

#### Package Review Success Criteria

- [ ] COPR repository operational
- [ ] Package review approved
- [ ] Package imported to Fedora
- [ ] Available in Fedora repositories

---

### 4.4 Documentation for Packagers

**Priority**: LOW
**Estimated Effort**: Low

#### Packager Documentation Tasks

1. **Create PACKAGING.md**
   - Document build process
   - Document packaging for different distributions
   - Include sample spec file
   - Include packaging best practices

2. **Distribution Support**
   - Document Fedora/RHEL packaging
   - Consider Debian/Ubuntu packaging (optional)
   - Consider Arch Linux packaging (optional)
   - Document container image creation

#### Packager Documentation Success Criteria

- [ ] PACKAGING.md created
- [ ] Packaging process documented
- [ ] Easy for other packagers to contribute

---

## Timeline and Priorities

### Phase 1: Modernization (6-8 weeks)

1. **Week 1-2**: Go version update, dependency updates, code cleanup
2. **Week 2-3**: Linting and code quality improvements
3. **Week 3-4**: CI/CD modernization
4. **Week 4-5**: CLI improvements
5. **Week 5-8**: Documentation updates, man pages

### Phase 2: OCI Integration (6-8 weeks)

1. **Week 1-2**: OCI layout support and design
2. **Week 2-4**: oci-diff command implementation
3. **Week 4-6**: oci-patch command implementation
4. **Week 6-8**: Testing and refinement

### Phase 3: Testing (8-10 weeks)

1. **Week 1-3**: Unit test development (pkg/tar-diff)
2. **Week 3-4**: Unit test development (pkg/tar-patch)
3. **Week 4-5**: Unit test development (pkg/common, cmd/)
4. **Week 5-6**: OCI integration tests
5. **Week 6-7**: tmt test plan creation and FMF metadata
6. **Week 7-8**: Testing Farm integration and multi-platform testing
7. **Week 8-10**: Benchmark tests and optimization

### Phase 4: Fedora Packaging (4-8 weeks + review time)

1. **Week 1-2**: RPM spec creation and testing
2. **Week 2-3**: Build system integration
3. **Week 3-4**: COPR setup and testing
4. **Week 4+**: Fedora review process (variable timeline)

**Total Estimated Timeline**: 24-34 weeks (not including Fedora review wait time)

---

## Dependencies Between Phases

- **Phase 2 depends on Phase 1**: Need modern Go version and clean code before OCI work
- **Phase 3 depends on Phase 1 and 2**: Need working code to test
- **Phase 4 can partially overlap Phase 3**: Can create spec file early, but should have tests before submitting to Fedora
- **Recommended sequence**: Phase 1 → Phase 2 → Phase 3 → Phase 4

---

## Risk Assessment

### High Risk Items

1. **Dependency Updates**: May introduce breaking changes
   - *Mitigation*: Thorough testing, maintain compatibility matrix

2. **Test Development Time**: Creating comprehensive tests takes time
   - *Mitigation*: Prioritize critical paths, iterate incrementally

3. **OCI Integration Complexity**: Registry auth, layer formats, edge cases
   - *Mitigation*: Start with OCI layouts, expand incrementally

4. **Fedora Review Delays**: Review process can take weeks/months
   - *Mitigation*: Start COPR early, engage with community

### Medium Risk Items

1. **Go Version Jump**: Large jump from 1.14 to 1.25 may expose issues
   - *Mitigation*: Thorough testing, comprehensive test suite

2. **CI/CD Migration**: Transition from Cirrus to GitHub Actions
   - *Mitigation*: Test GitHub Actions workflows thoroughly before removing Cirrus

3. **Testing Farm Learning Curve**: Team may be unfamiliar with tmt/Testing Farm
   - *Mitigation*: Start with simple test plans, expand gradually

4. **Hardlink/Duplicate File TODOs**: May require significant code changes
   - *Mitigation*: Analyze impact early, document limitations if not feasible

### Low Risk Items

1. **Documentation**: Low technical risk
2. **Linting**: May find issues but easy to fix
3. **Spec File Creation**: Well-documented process
4. **CLI Improvements**: Low complexity additions

---

## Success Metrics

### Modernization Success

- ✅ Go version ≥ 1.25
- ✅ All dependencies at latest stable versions
- ✅ Cirrus CI removed, GitHub Actions fully operational
- ✅ Zero high-priority lint issues
- ✅ Multi-platform CI via GitHub Actions (Linux, macOS, Windows)
- ✅ Deprecated APIs replaced
- ✅ CLI typos fixed
- ✅ Man pages available

### OCI Integration Success

- ✅ Can diff two OCI layout directories
- ✅ Can reconstruct target image from base + diff
- ✅ Works with real Fedora images
- ✅ Diff is significantly smaller than full image

### Testing Success

- ✅ Code coverage ≥ 80%
- ✅ Unit tests for all packages
- ✅ OCI integration tests passing
- ✅ Benchmark suite operational
- ✅ Tests pass consistently on all platforms
- ✅ tmt test plans created and working
- ✅ Testing Farm integration operational
- ✅ Multi-architecture testing via Testing Farm

### Packaging Success

- ✅ Valid RPM spec file with man pages
- ✅ COPR repository operational
- ✅ Package available in Fedora
- ✅ Documented packaging process

---

## Resources Required

### Tools

- Go 1.25+ development environment
- Fedora development machine (VM acceptable)
- tmt installed locally for test development
- rpmbuild and mock for package building
- COPR account for testing builds
- Fedora Account System account
- Testing Farm API access (via GitHub integration)
- GitHub repository with Actions enabled
- skopeo and podman for OCI testing

### Documentation

- Fedora Go Packaging Guidelines
- Fedora Package Review Process
- Go testing best practices
- golangci-lint documentation
- tmt documentation (<https://tmt.readthedocs.io/>)
- Testing Farm documentation (<https://docs.testing-farm.io/>)
- FMF (Flexible Metadata Format) specification
- OCI Image Specification (<https://github.com/opencontainers/image-spec>)
- containers/image library documentation

### Time

- **Developer Time**: ~400-600 hours estimated
  - Modernization: 100-150 hours
  - OCI Integration: 100-150 hours
  - Testing: 160-240 hours (includes tmt/Testing Farm setup)
  - Packaging: 80-120 hours
- **Review Wait Time**: Variable (2-12 weeks typical)

---

## Next Steps

1. **Immediate Actions**:
   - Review and approve this plan
   - Set up development environment
   - Create project tracking (GitHub issues/milestones)

2. **Week 1 Focus**:
   - Update go.mod to Go 1.25
   - Fix deprecated `io/ioutil` usage
   - Fix CLI typos
   - Update Makefile
   - Remove Cirrus CI configuration

3. **Quick Wins**:
   - Create initial GitHub Actions workflow
   - Run gofmt on entire codebase
   - Add basic badges to README
   - Install tmt locally and explore

4. **Long-term Planning**:
   - Create GitHub milestones for each phase
   - Assign issues to milestones
   - Set up weekly progress reviews
   - Prototype OCI layout support early

---

## Appendices

### A. Fedora Packaging Resources

- Packaging Guidelines: <https://docs.fedoraproject.org/en-US/packaging-guidelines/>
- Go Packaging: <https://docs.fedoraproject.org/en-US/packaging-guidelines/Golang/>
- Package Review Process: <https://docs.fedoraproject.org/en-US/package-maintainers/Package_Review_Process/>
- COPR: <https://copr.fedorainfracloud.org/>

### B. Go Testing Resources

- Go Testing Package: <https://pkg.go.dev/testing>
- Table-Driven Tests: <https://github.com/golang/go/wiki/TableDrivenTests>
- Testify Framework: <https://github.com/stretchr/testify> (optional)

### C. CI/CD Resources

- GitHub Actions: <https://docs.github.com/en/actions>
- GitHub Actions for Go: <https://github.com/actions/setup-go>
- goreleaser: <https://goreleaser.com/>

### D. Test Management Resources

- tmt Documentation: <https://tmt.readthedocs.io/>
- tmt GitHub: <https://github.com/teemtee/tmt>
- Testing Farm: <https://testing-farm.io/>
- Testing Farm Documentation: <https://docs.testing-farm.io/>
- FMF Specification: <https://fmf.readthedocs.io/>
- Fedora CI Documentation: <https://docs.fedoraproject.org/en-US/ci/>

### E. OCI Resources

- OCI Image Specification: <https://github.com/opencontainers/image-spec>
- OCI Distribution Specification: <https://github.com/opencontainers/distribution-spec>
- containers/image library: <https://github.com/containers/image>
- skopeo: <https://github.com/containers/skopeo>
- podman: <https://github.com/containers/podman>

---

## Document Revision History

- **v1.0** - 2025-01-13 - Initial plan created
- **v1.1** - 2025-01-13 - Updated to remove Cirrus CI, add tmt/Testing Farm, target Go 1.25+
- **v1.2** - 2025-01-13 - Added OCI integration phase, code cleanup section, CLI improvements, man pages, known issues
- Claude assisted with plan creation and updates
