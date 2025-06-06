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

1. **Cross-Compiler Toolchain Only**: Install `gcc-aarch64-linux-gnu` and related tools without adding foreign architectures
2. **Host Dependencies**: Install build-time tools that run on the build system (x86_64): `lua5.1`, `cmake`, `ninja-build`
3. **Bundled Target Dependencies**: Use `-DUSE_BUNDLED=ON` to build all target dependencies (LuaJIT, libuv, etc.) from source for aarch64
4. **Static Linking**: Dependencies are built and linked statically, creating self-contained binaries
5. **CMake Toolchain File**: Specifies cross-compiler and target environment configuration
6. **No Target System Packages**: Avoids complex apt configuration and repository management

**Key Distinction**:
- **Host dependencies** (like `lua5.1`) run on the build machine during compilation
- **Target dependencies** (like LuaJIT) are compiled for the target architecture and bundled into the binary

This approach follows the **canonical cross-compilation pattern** documented in the referenced jensd.be article, where dependencies are built from source rather than installed as system packages.

### CMake Toolchain Configuration

```cmake
# Key directives for cross-compilation with bundled dependencies
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

# Critical: Disable all system package finding for cross-compilation
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)  # Use host programs
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY NEVER)  # Don't search for target libraries
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE NEVER)  # Don't search for target headers
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE NEVER)  # Don't search for target packages

# Force bundled dependencies by explicitly disabling system package finding
set(CMAKE_DISABLE_FIND_PACKAGE_LibUV TRUE)
set(CMAKE_DISABLE_FIND_PACKAGE_Luv TRUE)
set(CMAKE_DISABLE_FIND_PACKAGE_Msgpack TRUE)
# ... (additional packages disabled to force bundled builds)
```

### Dependency Management

**For amd64 builds** (system packages):
- `libuv1-dev` - Asynchronous I/O library
- `libmsgpackc-dev` - MessagePack serialization
- `libtermkey-dev` - Terminal key parsing
- `libvterm-dev` - Virtual terminal emulator
- `libluajit-5.1-dev` - LuaJIT runtime
- `libtree-sitter-dev` - Syntax parsing library

**For aarch64 builds** (bundled/built from source):
- All dependencies are built from source using `-DUSE_BUNDLED=ON`
- No target architecture system packages required
- Results in larger but self-contained binaries
- **Note**: Host Lua interpreter (`lua5.1`) is still needed on the build system for CMake scripts, while target LuaJIT is built from bundled sources

**Build Dependencies** (host architecture):
- `cmake`, `ninja-build` - Build system
- `gettext` - Internationalization
- `gcc-aarch64-linux-gnu` - Cross-compiler toolchain (aarch64 only)
- `lua5.1` - Host Lua interpreter needed for build process (aarch64 only)

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

**Symptom**: `Could NOT find Luv (missing: LUV_LIBRARY LUV_INCLUDE_DIR)` during cross-compilation
```bash
# Diagnosis: CMake still trying to find system packages despite USE_BUNDLED=ON
# Solution: Use CMAKE_DISABLE_FIND_PACKAGE_* in toolchain file to force bundled builds
# Verify toolchain configuration:
grep CMAKE_DISABLE_FIND_PACKAGE aarch64-toolchain.cmake  # Should show disabled packages
```

**Symptom**: `Failed to find a Lua 5.1-compatible interpreter`
```bash
# Diagnosis: Missing host Lua interpreter for build process
# Solution: Install Lua interpreter on build system (not target system)
sudo apt-get install lua5.1
# Note: This is separate from target LuaJIT which comes from bundled dependencies
```

**Symptom**: `Package 'lua' not found` during cross-compilation
```bash
# Diagnosis: Missing bundled dependency configuration
# Solution: Ensure -DUSE_BUNDLED=ON is set for cross-compilation
cmake -DUSE_BUNDLED=ON ...
```

**Symptom**: `undefined reference to symbol` during linking
```bash
# Diagnosis: Cross-compiler not finding proper libraries
# Solution: Verify CMake toolchain file configuration
aarch64-linux-gnu-gcc --version  # Confirm toolchain available
```

**Symptom**: CMake fails to detect cross-compilation environment
```bash
# Diagnosis: Toolchain file not properly configured
# Solution: Verify CMAKE_SYSTEM_NAME and CMAKE_SYSTEM_PROCESSOR in toolchain file
grep "CMAKE_SYSTEM" aarch64-toolchain.cmake
```

**Symptom**: Binary compiled for wrong architecture
```bash
# Diagnosis: Cross-compiler not being used
# Solution: Verify CMAKE_C_COMPILER and CMAKE_CXX_COMPILER settings
file build/bin/nvim  # Should show ARM aarch64, not x86-64
```

### Build Environment Debugging

```bash
# Verify cross-compiler availability
aarch64-linux-gnu-gcc --version

# Check CMake toolchain configuration
cat aarch64-toolchain.cmake | grep CMAKE_C_COMPILER
cat aarch64-toolchain.cmake | grep CMAKE_SYSTEM_PROCESSOR
cat aarch64-toolchain.cmake | grep CMAKE_DISABLE_FIND_PACKAGE

# Verify bundled dependencies configuration
cmake -L build/ | grep USE_BUNDLED  # Should show ON for cross-compilation

# Test cross-compilation simple program
echo 'int main(){return 0;}' > test.c
aarch64-linux-gnu-gcc test.c -o test.arm64
file test.arm64  # Should show ARM aarch64

# Check CMake detection and package disabling
cmake -B test-build -DCMAKE_TOOLCHAIN_FILE=aarch64-toolchain.cmake
grep "CMAKE_SYSTEM_PROCESSOR" test-build/CMakeCache.txt  # Should show aarch64
grep "CMAKE_DISABLE_FIND_PACKAGE" aarch64-toolchain.cmake  # Should show disabled packages

# Verify no system packages are being found (good for bundled builds)
cmake -B test-build -DCMAKE_TOOLCHAIN_FILE=aarch64-toolchain.cmake --debug-find
# Should not show system package paths being searched
```

### Performance Considerations

- **Build Time**:
  - amd64 (system deps): ~8-12 minutes
  - aarch64 (bundled deps): ~20-30 minutes (compiles all dependencies from source)
- **Artifact Size**:
  - amd64: ~8-12MB .deb package (dynamic linking)
  - aarch64: ~15-25MB .deb package (static/bundled dependencies)
- **Memory Usage**: ~3-4GB peak during cross-compilation (building multiple dependencies)

## References

**Primary Sources:**
- [Neovim Build Documentation](https://github.com/neovim/neovim/wiki/Building-Neovim)
- [Cross-compiling for ARM/aarch64 on Debian/Ubuntu](https://jensd.be/1126/linux/cross-compiling-for-arm-or-aarch64-on-debian-or-ubuntu) - **Key reference for bundled dependency approach**
- [CMake Cross Compilation](https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html#cross-compiling)
- [Debian Multi-Arch](https://wiki.debian.org/Multiarch/HOWTO)

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
