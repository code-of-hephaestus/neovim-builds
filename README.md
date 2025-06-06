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
runs-on: blacksmith-4vcpu-ubuntu-2404
```

#### ARM64 Workflow
```yaml
runs-on: [blacksmith-4vcpu-ubuntu-2404-arm, linux, arm64]
```

The ARM64 runner specification uses array notation to ensure proper job allocation to ARM64-capable Blacksmith infrastructure.

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

#### 4. Binary Verification (ARM64 Only)
The ARM64 workflow includes explicit architecture verification:
```bash
file bin/nvim | grep -q "ARM aarch64" && echo "✓ aarch64 binary verified"
```

#### 5. Package Validation (ARM64 Only)
Performs installation testing to verify package integrity:
```bash
sudo dpkg -i nvim-stable-linux-aarch64.deb
sudo apt-get install -f -y  # Dependency resolution
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

### Workflow Variations

| Aspect | AMD64 | ARM64 |
|--------|-------|--------|
| Binary Verification | ❌ | ✅ |
| Installation Testing | ❌ | ✅ |
| Release Body | Upstream release notes | Enhanced with build metadata |
| Action Versions | `@v1` (upload-release-asset) | `@v2` (action-gh-release) |

### ARM64 Enhanced Features

The ARM64 workflow implements additional verification stages due to cross-compilation complexity and provides enhanced release documentation with installation instructions and verification commands.

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
- **Blacksmith Runners**: Architecture-specific compute infrastructure

## Execution Context

These workflows operate under manual dispatch model, requiring explicit user initiation. The architecture-specific builds ensure native compilation performance while maintaining consistent packaging standards across both x86_64 and aarch64 target platforms.

The implementation follows a contract-first approach with explicit error propagation and defensive verification stages, particularly evident in the ARM64 workflow's comprehensive validation pipeline.
