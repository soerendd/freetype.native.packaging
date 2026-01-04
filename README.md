# FreeType Build Workflow

This GitHub Actions workflow builds FreeType for Windows and Linux platforms.

## Windows Build Configuration

The Windows build is configured to produce a **standalone DLL with all dependencies statically linked**:

- **Static Runtime Linking**: The Visual C++ runtime is statically linked (`/MT` flag) so the DLL doesn't require external runtime DLLs (e.g., `vcruntime140.dll`, `msvcp140.dll`)
- **Static Dependencies**: All optional dependencies (zlib, bzip2, PNG, Brotli) are built as static libraries using vcpkg and linked into the FreeType DLL
- **No External DLLs Required**: The resulting DLL only depends on system libraries (kernel32.dll, etc.)
- **Architecture Support**: Both x64 and Win32 (x86) builds
- **HarfBuzz**: Currently disabled (can be enabled if needed)

### Build Artifacts

**Windows (x64 and Win32)**:
- `freetype.dll` - Main shared library
- `freetype.lib` - Import library for linking
- Configuration headers

**Linux (x64)**:
- `libfreetype.so*` - Shared library with statically linked dependencies
- Configuration headers

## Features Included

With static linking, the binaries include support for:
- ✅ **zlib**: Gzip-compressed font data
- ✅ **bzip2**: BZip2-compressed fonts
- ✅ **PNG**: PNG-compressed embedded bitmaps (color emoji fonts)
- ✅ **Brotli**: WOFF2 web font format support
- ❌ **HarfBuzz**: Disabled (can be enabled if needed for advanced text shaping)

## Usage

The workflow runs automatically on:
- Push to `main` or `master` branches
- Pull requests to `main` or `master` branches
- Manual trigger via GitHub Actions UI

Artifacts are available for download after each workflow run.

## Verification

**Windows**: Uses `dumpbin` to:
1. Check the DLL's external dependencies (should only show system DLLs like `KERNEL32.dll`, `USER32.dll`)
2. Verify static runtime linkage (no `vcruntime140.dll` or `msvcp140.dll`)

**Linux**: Uses `ldd` to check shared library dependencies and `nm` to verify static symbols are embedded

## Manual Build

To build locally with the same configuration:

### Windows

First, install dependencies with vcpkg:
```powershell
git clone https://github.com/microsoft/vcpkg.git
.\vcpkg\bootstrap-vcpkg.bat
.\vcpkg\vcpkg install zlib:x64-windows-static bzip2:x64-windows-static libpng:x64-windows-static brotli:x64-windows-static
```

Then build FreeType:
```powershell
# Set vcpkg triplet via environment variable
$env:VCPKG_DEFAULT_TRIPLET = "x64-windows-static"

cmake -B build -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_TOOLCHAIN_FILE="$PWD/vcpkg/scripts/buildsystems/vcpkg.cmake" `
  -DBUILD_SHARED_LIBS=ON `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_MSVC_RUNTIME_LIBRARY="MultiThreaded" `
  -DFT_REQUIRE_ZLIB=TRUE `
  -DFT_REQUIRE_BZIP2=TRUE `
  -DFT_REQUIRE_PNG=TRUE `
  -DFT_REQUIRE_BROTLI=TRUE `
  -DFT_DISABLE_HARFBUZZ=TRUE

cmake --build build --config Release
```

**Notes**: 
- The toolchain file must be specified with `-DCMAKE_TOOLCHAIN_FILE` (no space after `-D`)
- The vcpkg triplet is set via the `VCPKG_DEFAULT_TRIPLET` environment variable
- You may see a harmless warning "Could NOT find PkgConfig" - this is safe to ignore
- If you get build errors, clear the CMake cache: `rm -r build` and reconfigure

### Linux

Dependencies must be built as static libraries first. The workflow automates this, but for manual builds, you can use your distribution's static library packages if available, or build from source with `-fPIC` flag.

```bash
cmake -B build \
  -D CMAKE_BUILD_TYPE=Release \
  -D BUILD_SHARED_LIBS=ON \
  -D CMAKE_PREFIX_PATH=/path/to/static/deps \
  -D FT_REQUIRE_ZLIB=TRUE \
  -D FT_REQUIRE_BZIP2=TRUE \
  -D FT_REQUIRE_PNG=TRUE \
  -D FT_REQUIRE_BROTLI=TRUE \
  -D FT_DISABLE_HARFBUZZ=TRUE

cmake --build build
```

## Build Time

- **Windows**: ~10-15 minutes (includes vcpkg dependency installation)
- **Linux**: ~15-20 minutes (includes building dependencies from source)
