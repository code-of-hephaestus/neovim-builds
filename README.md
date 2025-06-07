# Neovim .deb Package Build Workflows

This repository contains GitHub Actions workflows for building native Debian packages (.deb) of Neovim from the upstream stable branch, targeting both AMD64 and ARM64 architectures.

## Architecture Overview

The build system implements a dual-architecture strategy with architecture-specific runners to ensure native compilation and optimal binary performance:

- **AMD64 Build**: Executes on standard Blacksmith 4vCPU Ubuntu 24.04 runners
- **ARM64 Build**: Executes on Blacksmith ARM64-specific runners (`blacksmith-4vcpu-ubuntu-2404-arm`)

Both workflows follow identical build methodologies with architecture-specific adaptations for package naming and verification.

## Workflow Specifications

### Build Triggers
- **Dispatch Model**: Manual workflow execution via `workflow_dispatch`
- **Concurrency Control**: Single workflow instance per ref with `cancel-in-progress: true`
- **Timeout Boundary**: 60-minute execution limit per workflow

### Runner Infrastructure

#### AMD64 Workflow
```yaml
runs-on: blacksmith-32vcpu-ubuntu-2404
```

#### ARM64 Workflow
```yaml
runs-on: blacksmith-32vcpu-ubuntu-2404-arm
```

### Build Process Pipeline

#### 1. Prerequisites Installation
Installs canonical build toolchain and Debian packaging utilities:
```bash
# Core build dependencies
ninja-build gettext cmake unzip curl

# Debian packaging toolchain
build-essential binutils lintian debhelper dh-make devscripts
```

#### 2. Source Acquisition & Compilation
- Clones upstream Neovim repository from `https://github.com/neovim/neovim`
- Checks out `stable` branch for predictable release artifacts
- Executes release build: `make CMAKE_BUILD_TYPE=Release`
- Generates architecture-specific .deb package via CPack

#### 3. Release Metadata Extraction
Dynamically extracts version information from GitHub API:
```bash
# Version parsing with jq and regex extraction
version_number=$(curl -s https://api.github.com/repos/neovim/neovim/releases/tags/stable | \
  jq -r '.body' | head -5 | grep -oE "v[0-9]+.[0-9].+[0-9]+")
```

#### 4. Binary Verification
Both workflows include explicit architecture verification:
```bash
# AMD64 verification
file bin/nvim | grep -q "x86-64" && echo "✓ amd64 binary verified"

# ARM64 verification
file bin/nvim | grep -q "ARM aarch64" && echo "✓ aarch64 binary verified"
```

#### 5. Package Validation
Both workflows perform installation testing to verify package integrity:
```bash
# Architecture-specific package installation with automatic dependency resolution
sudo apt install -y ./nvim-stable-linux-{amd64|aarch64}.deb
which nvim && nvim --version  # Functionality verification
```

#### 6. Release Asset Preparation
Workflows rename package files to include version information before upload:
```bash
# Create versioned filename for release asset
versioned_filename="nvim-{version}-stable-linux-{arch}.deb"
cp nvim-stable-linux-{arch}.deb "$versioned_filename"
```

### Release Artifact Generation

Both workflows create GitHub releases with systematic naming conventions:

**AMD64**: `nvim-{version}-stable-linux-amd64`
**ARM64**: `nvim-{version}-stable-linux-aarch64`

### Package Naming Schema
- **AMD64**: `nvim-stable-linux-amd64.deb`
- **ARM64**: `nvim-stable-linux-aarch64.deb`

The final release assets maintain version-prefixed naming for disambiguation.

## Implementation Differences

### Workflow Standardization

Both workflows now implement identical feature sets with architecture-specific adaptations:

| Aspect | AMD64 | ARM64 |
|--------|-------|--------|
| Binary Verification | ✅ (`x86-64`) | ✅ (`ARM aarch64`) |
| Installation Testing | ✅ (`apt install`) | ✅ (`apt install`) |
| Asset Preparation | ✅ (Version-named files) | ✅ (Version-named files) |
| Release Body | Enhanced with build metadata | Enhanced with build metadata |
| Action Versions | `@v2` (action-gh-release) | `@v2` (action-gh-release) |
| Runner Specification | 32vCPU Blacksmith | 32vCPU Blacksmith ARM |

### Enhanced Verification Pipeline

Both workflows implement comprehensive validation stages:
- **Architecture Verification**: Binary inspection via `file` command with architecture-specific pattern matching
- **Installation Testing**: Package installation validation using `sudo apt install` for automatic dependency resolution
- **Asset Preparation**: File renaming to include version information in release artifacts
- **Enhanced Documentation**: Standardized release notes with installation instructions and verification commands

## Technical Dependencies

### Required Permissions
```yaml
permissions:
  contents: write  # Release creation and asset upload
```

### Environment Variables
```yaml
ACTIONS_RUNNER_DEBUG: true
ACTIONS_STEP_DEBUG: true
```

Debug flags enabled for comprehensive workflow telemetry during execution.

### External Dependencies
- **GitHub API**: Release metadata extraction
- **jq**: JSON parsing for version extraction
- **CPack**: Debian package generation
- **Blacksmith Runners**: High-performance 32vCPU architecture-specific compute infrastructure
- **GitHub Actions**: `softprops/action-gh-release@v2` for unified release management

### Workflow Modernization

Both workflows implement current GitHub Actions best practices:
- **Environment Files**: Uses `$GITHUB_OUTPUT` instead of deprecated `::set-output` commands
- **Unified Release Management**: Single action handles both release creation and asset upload
- **Enhanced Package Installation**: Uses `sudo apt install` for automatic dependency resolution
- **Asset Naming**: Version-specific filenames for release artifacts
- **Defensive Validation**: Comprehensive verification pipeline with explicit error propagation

## Execution Context

These workflows operate under manual dispatch model, requiring explicit user initiation. The architecture-specific builds ensure native compilation performance while maintaining consistent packaging standards across both x86_64 and aarch64 target platforms.

Both implementations follow a contract-first approach with explicit error propagation and defensive verification stages, implementing comprehensive validation pipelines that verify binary architecture, test package installation, and provide enhanced release documentation with installation instructions and verification commands.
