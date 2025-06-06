# Neovim Multi-Architecture Build Pipeline

**Automated GitHub Actions workflow for building Neovim .deb packages targeting amd64 and aarch64 architectures.**

## Architecture Overview

This workflow implements **cross-compilation** for Neovim using the canonical Ubuntu toolchain, following the [official Neovim build documentation](https://github.com/neovim/neovim/wiki/Building-Neovim) with architecture-specific extensions.

### Key Technical Components

- **Host Environment**: Ubuntu 24.04 GitHub runner (amd64)
- **Target Architectures**: amd64 (native), aarch64 (cross-compiled)
- **Toolchain**: GCC cross-compilation suite (`gcc-aarch64-linux-gnu`)
- **Build System**: CMake with custom toolchain files
- **Package Format**: Debian .deb packages via CPack
- **Dependency Strategy**: System packages from Ubuntu Ports (arm64) repositories

## Usage

### Manual Trigger

1. Navigate to **Actions** → **Build Neovim .deb Package (multi-arch)**
2. Click **Run workflow**
3. Select target architecture:
   - `amd64`: Native compilation for x86-64
   - `aarch64`: Cross-compilation for ARM64

### Automated Integration

```yaml
# Example: Trigger from another workflow
- name: Build Neovim aarch64
  uses: ./.github/workflows/neovim-build.yml
  with:
    arch: aarch64
```

## Technical Implementation

### Cross-Compilation Strategy

**For aarch64 builds:**

1. **Sysroot Setup**: Ubuntu Ports repositories provide pre-built ARM64 dependencies
2. **Toolchain Configuration**: CMake toolchain file specifies cross-compiler and target environment
3. **PKG-Config**: Environment configured for cross-compilation library discovery
4. **Dependency Resolution**: `dpkg --add-architecture arm64` enables multi-arch package installation

### CMake Toolchain Configuration

```cmake
# Key directives for cross-compilation
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

# Critical: Proper find path modes
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)  # Use host programs
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)  # Use target libraries
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)  # Use target headers
```

### Dependency Management

**Runtime Dependencies** (installed for target architecture):
- `libuv1-dev` - Asynchronous I/O library
- `libmsgpackc-dev` - MessagePack serialization
- `libtermkey-dev` - Terminal key parsing
- `libvterm-dev` - Virtual terminal emulator
- `libluajit-5.1-dev` - LuaJIT runtime (handles aarch64 assembly)
- `libtree-sitter-dev` - Syntax parsing library

**Build Dependencies** (host architecture):
- `cmake`, `ninja-build` - Build system
- `gettext` - Internationalization
- `gcc-aarch64-linux-gnu` - Cross-compiler toolchain
- `pkg-config` - Library discovery (configured via environment variables)

## Output Artifacts

### Package Naming Convention

```
nvim-stable-linux-{arch}.deb
```

**Examples:**
- `nvim-stable-linux-amd64.deb`
- `nvim-stable-linux-aarch64.deb`

### GitHub Release Structure

Each build creates a tagged release:

```
Tag: nvim-v{version}-stable-linux-{arch}
Name: Neovim v{version} (Linux {arch})
Assets: nvim-stable-linux-{arch}.deb
```

## Installation Instructions

### Direct Package Installation

```bash
# Download from releases
wget https://github.com/{your-org}/{your-repo}/releases/download/{tag}/nvim-stable-linux-aarch64.deb

# Install with dependency resolution
sudo dpkg -i nvim-stable-linux-aarch64.deb
sudo apt-get install -f  # Resolve any missing dependencies

# Verify installation
nvim --version
```

### Container Deployment

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y wget
RUN wget {release-url}/nvim-stable-linux-aarch64.deb && \
    dpkg -i nvim-stable-linux-aarch64.deb && \
    apt-get install -f -y
```

## Verification & Testing

### Binary Architecture Validation

```bash
# Verify target architecture
file nvim-stable-linux-aarch64.deb
# Expected: ARM aarch64

# Extract and inspect binary
dpkg-deb -x nvim-stable-linux-aarch64.deb extracted/
file extracted/usr/bin/nvim
# Expected: ELF 64-bit LSB executable, ARM aarch64
```

### Package Integrity

```bash
# Inspect package metadata
dpkg-deb --info nvim-stable-linux-aarch64.deb

# Check dependencies
dpkg-deb --field nvim-stable-linux-aarch64.deb Depends
```

## Troubleshooting

### Common Build Failures

**Symptom**: `404 Not Found` errors for arm64 packages from security.ubuntu.com
```bash
# Diagnosis: Default Ubuntu repositories don't serve arm64 packages
# Solution: Restrict default repos to amd64, use Ubuntu Ports for arm64 (already configured)
# Check repository configuration:
cat /etc/apt/sources.list  # Should show [arch=amd64]
cat /etc/apt/sources.list.d/arm64.list  # Should show [arch=arm64] with ports.ubuntu.com
```

**Symptom**: `E: Unable to locate package pkg-config-aarch64-linux-gnu`
```bash
# Diagnosis: Package name varies across Ubuntu versions
# Solution: Use environment variables in CMake toolchain (already configured)
# The PKG_CONFIG_* variables handle cross-compilation automatically
```

**Symptom**: `Package 'lua' not found`
```bash
# Diagnosis: Missing pkg-config setup
# Solution: Verify PKG_CONFIG_LIBDIR environment in toolchain file
```

**Symptom**: `undefined reference to symbol` during linking
```bash
# Diagnosis: Architecture mismatch in dependencies
# Solution: Ensure all :arm64 packages installed correctly
sudo apt-get install --reinstall libuv1-dev:arm64
```

**Symptom**: CPack fails with architecture errors
```bash
# Diagnosis: CPACK_DEBIAN_PACKAGE_ARCHITECTURE mismatch
# Solution: Verify CMake configuration sets correct architecture
cmake -DCPACK_DEBIAN_PACKAGE_ARCHITECTURE=arm64 ...
```

### Build Environment Debugging

```bash
# Verify cross-compiler availability
aarch64-linux-gnu-gcc --version

# Check repository configuration
cat /etc/apt/sources.list | head -5  # Should show [arch=amd64]
cat /etc/apt/sources.list.d/arm64.list  # Should show [arch=arm64] ports.ubuntu.com

# Validate architecture setup
dpkg --print-architecture  # Should show: amd64
dpkg --print-foreign-architectures  # Should show: arm64

# Check pkg-config paths
aarch64-linux-gnu-pkg-config --variable pc_path pkg-config 2>/dev/null || echo "Using environment variables"

# Validate installed dependencies
dpkg-query -l "*:arm64" | grep libuv
```

### Performance Considerations

- **Build Time**: aarch64 cross-compilation ~15-20 minutes
- **Artifact Size**: ~8-12MB .deb package
- **Memory Usage**: ~2GB peak during compilation

## References

**Primary Sources:**
- [Neovim Build Documentation](https://github.com/neovim/neovim/wiki/Building-Neovim)
- [CMake Cross Compilation](https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html#cross-compiling)
- [Debian Multi-Arch](https://wiki.debian.org/Multiarch/HOWTO)
- [Ubuntu Ports](https://wiki.ubuntu.com/ARM/RootfsFromScratch/BuildingComponents)

**Technical Specifications:**
- [ARM64 ABI](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst)
- [GCC Cross-Compilation](https://gcc.gnu.org/onlinedocs/gcc/Cross_002dcompilation.html)
- [CPack Debian Generator](https://cmake.org/cmake/help/latest/cpack_gen/deb.html)

## Contributing

### Adding New Architectures

1. Update workflow `options` in `workflow_dispatch`
2. Add architecture-specific toolchain installation
3. Create corresponding CMake toolchain file
4. Update package verification logic

### Dependency Updates

Monitor Ubuntu package availability:
```bash
# Check package availability across architectures
apt-cache madison libuv1-dev
apt-cache madison libuv1-dev:arm64
```

### Build Optimization

- **Parallel Builds**: Adjust `--parallel $(nproc)` for runner capacity
- **Caching**: Consider dependency caching for repeated builds
- **Matrix Builds**: Extend to build multiple architectures simultaneously
