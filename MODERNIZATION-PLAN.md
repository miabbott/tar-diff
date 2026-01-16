# tar-diff Modernization, Testing, and Fedora Packaging Plan

## Executive Summary

This document outlines a comprehensive plan to modernize the tar-diff
codebase, enhance testing coverage, and package the project for Fedora
Linux. The plan is organized into phases based on task priority and
effort, with CRITICAL and HIGH priority tasks appearing in early phases.
Low-effort tasks are prioritized within each priority level to enable
quick wins and incremental progress.

OCI container image integration is identified as a future development
opportunity and is intentionally deferred until the core modernization,
testing, and packaging work is complete.

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
4. **Outdated Makefile**: Uses `GO111MODULE="on"` (unnecessary since Go
   1.17)
5. **Limited OCI Integration**: Despite OCI being the primary use case,
   there's no direct OCI image support

### Testing Status

- **Unit Tests**: **NONE** (no *_test.go files exist)
- **Integration Tests**: Shell script (tests/test.sh, 83 lines)
- **Test Coverage**: Unknown (no unit tests to measure)
- **Benchmarks**: None

### Packaging Status

- **Fedora RPM**: No spec file
- **Container Images**: None
- **Installation**: Manual via `make install`

---

## Phase 1: Quick Wins

This phase focuses on HIGH priority tasks with LOW effort to build
momentum and establish a foundation for subsequent work.

### 1.1 Code Cleanup and Bug Fixes

**Priority**: HIGH
**Effort**: Low

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

#### Acceptance Criteria for Code Cleanup

- [ ] No deprecated `io/ioutil` usage
- [ ] All CLI typos fixed
- [ ] Makefile modernized
- [ ] TODOs addressed or documented as known limitations

---

### 1.2 Build System Integration

**Priority**: HIGH
**Effort**: Low

#### Build System Tasks

1. **Update Makefile**
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
     - tmt (for test management, optional)
   - Document runtime dependencies (minimal/none expected)

3. **Version Management**
   - Sync version between:
     - pkg/common/version.go
     - go.mod
     - tar-diff.spec (when created)
   - Consider using git tags for versioning
   - Automate version bumping

#### Acceptance Criteria for Build System

- [ ] `make dist` produces valid source tarball
- [ ] `make install` respects DESTDIR and PREFIX
- [ ] All dependencies documented
- [ ] Version synchronized across files

---

## Phase 2: Core Modernization

This phase addresses HIGH priority tasks with MEDIUM effort that form
the foundation of modernization.

### 2.1 Go Version and Module Updates

**Priority**: HIGH
**Effort**: Medium

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

#### Acceptance Criteria for Go Updates

- [ ] go.mod specifies Go 1.25 or later
- [ ] All dependencies updated to latest stable versions
- [ ] Code compiles without warnings on Go 1.25+
- [ ] All existing tests pass
- [ ] Code leverages modern Go idioms and features

---

### 2.2 Code Quality and Linting

**Priority**: HIGH
**Effort**: Medium

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

#### Acceptance Criteria for Linting

- [ ] golangci-lint v1.55+ configured and passing
- [ ] Zero high-priority linting issues
- [ ] All code formatted consistently
- [ ] Makefile includes `make lint` target

---

### 2.3 Windows Build Support

**Priority**: HIGH
**Effort**: Low

#### Current Windows Compatibility Issues

The codebase has several Unix-specific patterns that prevent successful
Windows builds:

1. **Hardcoded Unix paths**: `/var/tmp` used for temporary files
2. **Path separator issues**: Hardcoded `/` instead of `filepath.Join()`
3. **Package usage**: `path` package used instead of `filepath`
4. **Unix permission assumptions**: Octal permission bit checks

#### Windows Compatibility Tasks

1. **Fix Temporary File Handling**
   - File: `pkg/tar-diff/analysis.go:327`
   - Replace `ioutil.TempFile("/var/tmp", ...)` with
     `os.CreateTemp(os.TempDir(), ...)`

2. **Fix Path Concatenation**
   - File: `pkg/tar-patch/apply.go:71`
   - Replace `f.basePath + "/" + file` with
     `filepath.Join(f.basePath, file)`

3. **Replace `path` Package with `filepath`**
   - Files: `cmd/tar-diff/main.go`, `cmd/tar-patch/main.go`,
     `pkg/tar-diff/analysis.go`
   - Change import from `path` to `path/filepath`
   - Replace `path.Base()` with `filepath.Base()`
   - Replace `path.Clean()` with `filepath.Clean()`

4. **Fix Path Cleaning Functions**
   - Files: `pkg/tar-diff/analysis.go:68-75`, `pkg/tar-patch/apply.go:28-35`
   - Update `cleanPath()` to use `filepath.Clean()` without hardcoded
     leading slash
   - Ensure cross-platform path normalization

5. **Review Permission Bit Handling**
   - File: `pkg/tar-diff/analysis.go:102-104`
   - Document that Unix permission checks are for tar file analysis
   - Ensure behavior is correct when processing tar files on Windows
   - Consider adding platform-specific handling if needed

#### Acceptance Criteria for Windows Support

- [ ] Code compiles successfully with `GOOS=windows go build ./...`
- [ ] No hardcoded Unix paths remain
- [ ] All path operations use `filepath` package
- [ ] CI includes Windows build verification (see 2.4)
- [ ] Binary artifacts produced for Windows (amd64, arm64)

---

### 2.4 CI/CD Modernization

**Priority**: HIGH
**Effort**: Medium

#### CI/CD Tasks

1. **Remove Cirrus CI**
   - Remove `.cirrus.yml` configuration file (currently using Fedora 32,
     EOL)
   - Migrate all CI functionality to GitHub Actions
   - Archive/document any Cirrus-specific workflows

2. **GitHub Actions Workflows**
   - Create `.github/workflows/ci.yml` for continuous integration:
     - Build and test on Linux (Fedora 42/43/Rawhide, Ubuntu latest)
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

#### Acceptance Criteria for CI/CD

- [ ] Cirrus CI configuration removed
- [ ] GitHub Actions workflows fully operational
- [ ] Multi-platform testing (Linux, macOS, Windows)
- [ ] Code coverage tracking enabled with codecov.io
- [ ] Automated security scanning active
- [ ] All CI runs complete in <10 minutes
- [ ] CI badge displayed in README.md

---

## Phase 3: Testing Foundation

This phase addresses the CRITICAL testing gap and establishes proper
test infrastructure.

### 3.1 Unit Test Development

**Priority**: CRITICAL
**Effort**: High

#### Current Gap

**ZERO** unit tests exist in the codebase. This is the highest priority
item.

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

#### Acceptance Criteria for Unit Tests

- [ ] All packages have comprehensive unit tests
- [ ] Code coverage >= 80% overall
- [ ] Critical paths have >= 90% coverage
- [ ] All tests pass consistently
- [ ] Tests complete in reasonable time (<30s for unit tests)

---

### 3.2 Test Management with tmt and Testing Farm

**Priority**: HIGH
**Effort**: Medium

#### tmt Overview

Integrate [tmt (Test Management Tool)](https://github.com/teemtee/tmt)
and [Testing Farm](https://testing-farm.io/) for comprehensive test
orchestration and execution across multiple platforms and Fedora versions.

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
     execute:
       how: tmt
     ```

   - Create individual test plans for:
     - Unit tests (`plans/unit.fmf`)
     - Integration tests (`plans/integration.fmf`)
     - RPM package tests (`plans/rpm.fmf`)
     - Multi-architecture tests (`plans/multiarch.fmf`)

2. **FMF Metadata for Tests**
   - Create `.fmf/version` file to enable FMF
   - Add metadata to existing tests:
     - Create `tests/unit/main.fmf` for Go unit tests
     - Create `tests/integration/main.fmf` for integration tests
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

   - Create helper scripts in `tests/lib/`:
     - `setup.sh` - Test environment preparation
     - `helpers.sh` - Common test utilities
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

#### Acceptance Criteria for tmt Integration

- [ ] tmt test plans created and validated
- [ ] FMF metadata added to all tests
- [ ] Testing Farm integration operational
- [ ] Tests run automatically on PRs via Testing Farm
- [ ] Multi-platform and multi-arch testing working
- [ ] Test results visible in GitHub PR checks
- [ ] Local tmt test execution working
- [ ] Documentation for running tests with tmt

---

## Phase 4: Fedora Packaging

This phase focuses on creating and submitting the Fedora package.

### 4.1 RPM Spec File Creation

**Priority**: HIGH
**Effort**: Medium

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
     - Update to 0.2.0 with modernization
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

4. **Makefile RPM Targets**
   - Add `make srpm` target for source RPM creation
   - Add `make rpm` target for binary RPM creation
   - Integrate with the spec file created above

#### Acceptance Criteria for RPM Spec

- [ ] Valid RPM spec file created
- [ ] Spec follows Fedora guidelines
- [ ] Builds successfully with `rpmbuild`
- [ ] `make srpm` produces valid SRPM
- [ ] `make rpm` produces valid RPM
- [ ] Installs correctly with `dnf`
- [ ] Man pages included

---

### 4.2 Fedora Package Review Process

**Priority**: MEDIUM
**Effort**: High (includes waiting time)

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

#### Acceptance Criteria for Package Review

- [ ] COPR repository operational
- [ ] Package review approved
- [ ] Package imported to Fedora
- [ ] Available in Fedora repositories

---

## Phase 5: Enhancements

This phase addresses MEDIUM priority tasks that improve usability and
documentation.

### 5.1 CLI Improvements

**Priority**: MEDIUM
**Effort**: Medium

#### CLI Enhancement Tasks

1. **Add Progress and Verbose Output**
   - Add `-v` / `--verbose` flag for detailed operation logging
   - Show progress for large file operations
   - Display statistics after completion (files processed, compression
     ratio, etc.)

2. **Improve Output Options**
   - Add stdout support for `tar-diff` (currently only `tar-patch`
     supports `-`)
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

#### Acceptance Criteria for CLI

- [ ] Verbose mode available for all commands
- [ ] Progress indicators for large operations
- [ ] Consistent flag naming across commands
- [ ] Informational commands working

---

### 5.2 Documentation Updates

**Priority**: MEDIUM
**Effort**: Medium

#### Documentation Tasks

1. **Update README.md**
   - Add badges (build status, coverage, Go version, license)
   - Expand installation instructions (go install, package managers,
     binary releases)
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

#### Acceptance Criteria for Documentation

- [ ] README.md updated with modern badges
- [ ] All exported symbols documented
- [ ] Documentation visible on pkg.go.dev
- [ ] Man pages created and installable
- [ ] CONTRIBUTING.md and CHANGELOG.md present

---

### 5.3 Benchmark Tests

**Priority**: MEDIUM
**Effort**: Medium

#### Benchmark Tasks

1. **Create Benchmark Suite**
   - Benchmark diff generation for various file sizes
   - Benchmark patch application
   - Benchmark rolling checksum computation
   - Benchmark compression/decompression
   - Benchmark bsdiff for different similarity levels

2. **Performance Baseline**
   - Document current performance characteristics
   - Create benchmark comparison against previous versions
   - Track performance regression in CI
   - Compare with alternative tools (if applicable)

3. **Optimization Opportunities**
   - Profile code to identify bottlenecks
   - Consider memory optimizations
   - Evaluate parallel processing opportunities

#### Acceptance Criteria for Benchmarks

- [ ] Comprehensive benchmark suite
- [ ] Performance baseline documented
- [ ] CI tracks performance regressions

---

### 5.4 Test Infrastructure

**Priority**: MEDIUM
**Effort**: Low

#### Test Infrastructure Tasks

1. **Local Development Testing**
   - Ensure tests can run locally without tmt
   - Maintain `make test` for quick local validation
   - Document both tmt and direct test execution

2. **Test Utilities**
   - Create helper functions for test setup/teardown
   - Create tar generation utilities in `tests/lib/`
   - Create comparison and validation utilities

3. **CI Test Matrix**
   - GitHub Actions: Go 1.25+ on Linux, macOS, Windows
   - Testing Farm: Multiple Fedora versions and architectures
   - Both CI systems should run on every PR
   - Fast feedback from GitHub Actions (<5 min)
   - Comprehensive feedback from Testing Farm (<30 min)

#### Acceptance Criteria for Test Infrastructure

- [ ] Reusable test fixtures available
- [ ] Test utilities reduce duplication
- [ ] Multi-platform test coverage (GitHub Actions + Testing Farm)
- [ ] Clear documentation for both CI systems

---

## Phase 6: Optional Tasks

This phase addresses LOW priority tasks that can be completed as time
permits.

### 6.1 Documentation for Packagers

**Priority**: LOW
**Effort**: Low

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

#### Acceptance Criteria for Packager Docs

- [ ] PACKAGING.md created
- [ ] Packaging process documented
- [ ] Easy for other packagers to contribute

---

## Future Development: OCI Container Image Integration

**Note**: This section describes future development opportunities that
should be pursued after the core modernization, testing, and packaging
work is complete.

### Rationale for Deferral

While OCI image distribution is the primary use case for tar-diff, the
following factors support deferring this work:

1. The existing tar-diff functionality works correctly for individual
   tar files
2. OCI integration adds significant complexity (registry auth, manifest
   handling, etc.)
3. Core modernization and testing should be completed first to establish
   a solid foundation
4. Fedora packaging can proceed without OCI-specific features

### Future OCI Integration Scope

When the team is ready to pursue OCI integration, the following areas
should be addressed:

#### OCI Image Support Design

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

#### OCI Integration Tasks

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

#### Advanced OCI Tasks

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

#### OCI-Specific Testing

When OCI features are implemented, the following tests should be added:

1. **Real Image Testing**
   - Test with actual OCI image layers from popular base images
   - Test Fedora image version upgrades (e.g., fedora:40 -> fedora:41)
   - Test Alpine image updates (small, simple layers)
   - Test application images (compiled binaries)

2. **Layer Characteristic Testing**
   - Test with layers containing many small files (RPM-based images)
   - Test with layers containing few large files (compiled applications)
   - Test with layers near the 192MB bsdiff threshold

3. **Compression Format Testing**
   - Test cross-format: diff gzip-compressed, reconstruct for zstd
     validation
   - Test with zstd-compressed layers (increasingly common)
   - Test with uncompressed layers

4. **Edge Cases**
   - Test empty layers
   - Test layers with only metadata changes
   - Test images with many layers (>10)
   - Test layer reordering scenarios

---

## Timeline and Priorities

### Phase 1: Quick Wins (1-2 weeks)

1. **Week 1**: Code cleanup, bug fixes, Makefile modernization
2. **Week 1-2**: Build system integration

### Phase 2: Core Modernization (4-6 weeks)

1. **Week 1-2**: Go version update, dependency updates
2. **Week 2-3**: Linting and code quality improvements
3. **Week 3-6**: CI/CD modernization (GitHub Actions)

### Phase 3: Testing Foundation (6-8 weeks)

1. **Week 1-4**: Unit test development (highest priority)
2. **Week 4-6**: tmt test plan creation and FMF metadata
3. **Week 6-8**: Testing Farm integration and multi-platform testing

### Phase 4: Fedora Packaging (4-8 weeks + review time)

1. **Week 1-2**: RPM spec creation and testing
2. **Week 2-4**: COPR setup and testing
3. **Week 4+**: Fedora review process (variable timeline)

### Phase 5: Enhancements (4-6 weeks)

1. **Week 1-2**: CLI improvements
2. **Week 2-4**: Documentation updates
3. **Week 4-6**: Benchmark tests and test infrastructure

### Phase 6: Optional Tasks (as time permits)

1. Packager documentation

**Total Estimated Timeline**: 19-30 weeks (not including Fedora review
wait time)

---

## Dependencies Between Phases

- **Phase 2 depends on Phase 1**: Quick wins establish a clean foundation
- **Phase 3 depends on Phase 2**: Need modern Go version and CI before
  comprehensive testing
- **Phase 4 can partially overlap Phase 3**: Can create spec file early,
  but should have tests before submitting to Fedora
- **Phase 5 can partially overlap Phase 4**: Enhancements can proceed
  during Fedora review
- **Future OCI work depends on Phases 1-4**: Core modernization must be
  complete first

---

## Risk Assessment

### High Risk Items

1. **Dependency Updates**: May introduce breaking changes
   - *Mitigation*: Thorough testing, maintain compatibility matrix

2. **Test Development Time**: Creating comprehensive tests takes time
   - *Mitigation*: Prioritize critical paths, iterate incrementally

3. **Fedora Review Delays**: Review process can take weeks/months
   - *Mitigation*: Start COPR early, engage with community

### Medium Risk Items

1. **Go Version Jump**: Large jump from 1.14 to 1.25 may expose issues
   - *Mitigation*: Thorough testing, comprehensive test suite

2. **CI/CD Migration**: Transition from Cirrus to GitHub Actions
   - *Mitigation*: Test GitHub Actions workflows thoroughly before
     removing Cirrus

3. **Testing Farm Learning Curve**: Team may be unfamiliar with
   tmt/Testing Farm
   - *Mitigation*: Start with simple test plans, expand gradually

4. **Hardlink/Duplicate File TODOs**: May require significant code
   changes
   - *Mitigation*: Analyze impact early, document limitations if not
     feasible

### Low Risk Items

1. **Documentation**: Low technical risk
2. **Linting**: May find issues but easy to fix
3. **Spec File Creation**: Well-documented process
4. **CLI Improvements**: Low complexity additions

---

## Success Metrics

### Modernization Success

- Go version >= 1.25
- All dependencies at latest stable versions
- Cirrus CI removed, GitHub Actions fully operational
- Zero high-priority lint issues
- Multi-platform CI via GitHub Actions (Linux, macOS, Windows)
- Deprecated APIs replaced
- CLI typos fixed
- Man pages available

### Testing Success

- Code coverage >= 80%
- Unit tests for all packages
- Benchmark suite operational
- Tests pass consistently on all platforms
- tmt test plans created and working
- Testing Farm integration operational
- Multi-architecture testing via Testing Farm

### Packaging Success

- Valid RPM spec file with man pages
- COPR repository operational
- Package available in Fedora
- Documented packaging process

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

### Documentation

- Fedora Go Packaging Guidelines
- Fedora Package Review Process
- Go testing best practices
- golangci-lint documentation
- tmt documentation (<https://tmt.readthedocs.io/>)
- Testing Farm documentation (<https://docs.testing-farm.io/>)
- FMF (Flexible Metadata Format) specification

### Time

- **Developer Time**: ~300-450 hours estimated
  - Quick Wins: 20-30 hours
  - Core Modernization: 80-120 hours
  - Testing: 120-180 hours (includes tmt/Testing Farm setup)
  - Packaging: 60-80 hours
  - Enhancements: 40-60 hours
- **Review Wait Time**: Variable (2-12 weeks typical)

---

## Next Steps

1. **Immediate Actions**:
   - Review and approve this plan
   - Set up development environment
   - Create project tracking (GitHub issues/milestones)

2. **Week 1 Focus**:
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

---

## Appendices

### A. Fedora Packaging Resources

- Packaging Guidelines:
  <https://docs.fedoraproject.org/en-US/packaging-guidelines/>
- Go Packaging:
  <https://docs.fedoraproject.org/en-US/packaging-guidelines/Golang/>
- Package Review Process:
  <https://docs.fedoraproject.org/en-US/package-maintainers/Package_Review_Process/>
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

---

## Document Revision History

- **v1.0** - 2025-01-13 - Initial plan created
- **v1.1** - 2025-01-13 - Updated to remove Cirrus CI, add tmt/Testing
  Farm, target Go 1.25+
- **v1.2** - 2025-01-13 - Added OCI integration phase, code cleanup
  section, CLI improvements, man pages, known issues
- **v2.0** - 2025-01-16 - Reorganized phases by priority and effort;
  deferred OCI integration to future development
