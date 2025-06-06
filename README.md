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
2. **Bundled Dependencies**: Use `-DUSE_BUNDLED=ON` to build all dependencies from source, eliminating the need for target architecture packages
3. **Static Linking**: Dependencies are built and linked statically, creating self-contained binaries
4. **CMake Toolchain File**: Specifies cross-compiler and target environment configuration
5. **No System Package Dependencies**: Avoids complex apt configuration and repository management

This approach follows the **canonical cross-compilation pattern** documented in the referenced jensd.be article, where dependencies are built from source rather than installed as system packages.

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

**Build Dependencies** (host architecture):
- `cmake`, `ninja-build` - Build system
- `gettext` - Internationalization
- `gcc-aarch64-linux-gnu` - Cross-compiler toolchain (aarch64 only)

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

# Verify bundled dependencies configuration
cmake -L build/ | grep USE_BUNDLED  # Should show ON for cross-compilation

# Test cross-compilation simple program
echo 'int main(){return 0;}' > test.c
aarch64-linux-gnu-gcc test.c -o test.arm64
file test.arm64  # Should show ARM aarch64

# Check CMake detection
cmake -B test-build -DCMAKE_TOOLCHAIN_FILE=aarch64-toolchain.cmake
grep "CMAKE_SYSTEM_PROCESSOR" test-build/CMakeCache.txt  # Should show aarch64
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
