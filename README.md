# toolchain-dist

This repository provides up to date builds of a minimal Windows C/C++ toolchain (Clang/LLD + BusyBox sh), ready to unpack and use. Built for the [lde](https://github.com/lde-org/lde) project.

## Supported Platforms

| Platform    | Architecture    | Download                                                                                                       |
| ----------- | --------------- | -------------------------------------------------------------------------------------------------------------- |
| **Windows** | x86-64          | [🔧 Toolchain](https://github.com/lde-org/toolchain-dist/releases/latest/download/toolchain-windows-x86-64.7z) \| [📦 7z](https://github.com/lde-org/toolchain-dist/releases/latest/download/7z-x86_64.exe) |
| **Windows** | aarch64 (ARM64) | [🔧 Toolchain](https://github.com/lde-org/toolchain-dist/releases/latest/download/toolchain-windows-aarch64.7z) \| [📦 7z](https://github.com/lde-org/toolchain-dist/releases/latest/download/7z-aarch64.exe) |

## Download

The latest builds are always available in the [latest release](https://github.com/lde-org/toolchain-dist/releases/latest).

## Toolchain Layout

After unpacking, the archive yields a single `toolchain/` directory:

```
toolchain/
├── bin/                          # Compiler, linker, binutils, make, shell
├── include/                      # C/C++ and Windows SDK headers
├── lib/                          # Clang/LLVM internal libraries
│   └── clang/22/
│       └── lib/windows/          # compiler-rt builtins (libclang_rt.builtins-*.a)
└── <target>/                     # Target sysroot (e.g. x86_64-w64-mingw32)
    ├── bin/                      # Runtime DLLs to deploy alongside built binaries
    └── lib/                      # Windows import libraries + CRT objects
```

### `bin/` — Build Tools

All tools are invoked from `toolchain/bin/`. Add this directory to `PATH` to use the toolchain.

| Tool | Provides | Notes |
|------|----------|-------|
| `gcc.exe` / `g++.exe` / `cc.exe` / `c++.exe` | C / C++ compiler | GCC-compat wrappers → Clang |
| `clang.exe` / `clang++.exe` | C / C++ compiler (native) | Direct entry point |
| `clang-22.exe` | Compiler backend | The actual compiler binary |
| `ar.exe` / `ranlib.exe` | Static library archiver | → `llvm-ar.exe` / `llvm-ranlib.exe` |
| `ld.lld.exe` | Linker (LLD) | Invoked via `ld` wrapper script |
| `ld` | Linker wrapper script | Wraps `ld.lld.exe` with correct flags |
| `dlltool.exe` | Import library / DEF generator | → `llvm-dlltool.exe` |
| `strip.exe` | Symbol stripper | → `llvm-strip.exe` |
| `objcopy.exe` | Object file manipulation | → `llvm-objcopy.exe` |
| `nm.exe` | Symbol table dumper | → `llvm-nm.exe` |
| `size.exe` | Section size reporter | → `llvm-size.exe` |
| `windres.exe` | Windows resource compiler | → `llvm-windres.exe` |
| `mingw32-make.exe` | GNU Make | Build automation |
| `sh.exe` | POSIX shell | BusyBox ash — used by LuaRocks configure scripts |
| `busybox.exe` | Multi-call binary | `sh.exe` is a copy of this |

### `include/` — Headers

Standard C, C++, and Windows SDK headers. Clang discovers them automatically relative to the `bin/` directory — no `-I` flags or environment variables needed.

### `lib/clang/22/lib/windows/` — Compiler Runtime

Contains `libclang_rt.builtins-*.a` (compiler-rt builtins for i386, x86_64, arm, aarch64). These provide low-level runtime routines (`__divdi3`, `__mulodi4`, etc.) and are linked automatically by Clang.

### `<target>/bin/` — Runtime DLLs

When a program is compiled and linked, it may depend on these DLLs at runtime. They must be placed alongside the built executable (or on `PATH`):

| DLL | Purpose |
|-----|---------|
| `libc++.dll` | C++ standard library |
| `libunwind.dll` | Stack unwinding (exception handling) |
| `libwinpthread-1.dll` | POSIX threads (pthreads) |

### Target triple

The toolchain's default target is:

| Architecture | Target triple |
|-------------|---------------|
| x86-64 | `x86_64-w64-mingw32` |
| aarch64 | `aarch64-w64-mingw32` |

This is what `gcc -dumpmachine` returns. Build systems (LuaRocks, autotools, cmake) discover the target automatically.

### Compiling a LuaRocks C module

With the toolchain on `PATH`, LuaRocks `build` / `make` commands work out of the box:

```
PATH = toolchain/bin;%PATH%
luarocks make foo-1.0-1.rockspec
```

The key tools LuaRocks depends on:

- `gcc` / `g++` → compilation
- `ld` → linking
- `ar` / `ranlib` → static libraries
- `sh` → running `./configure` scripts
- `make` → building (via `mingw32-make`)

### Runtime distribution

For lde to distribute compiled modules, the runtime DLLs must be included:

```
toolchain/<target>/bin/libc++.dll           →  bundle with .exe
toolchain/<target>/bin/libunwind.dll         →  bundle with .exe
toolchain/<target>/bin/libwinpthread-1.dll   →  bundle with .exe
```

These are the only DLLs needed at runtime; the toolchain's `bin/` DLLs (`libLLVM-22.dll`, `libclang-cpp.dll`) are only needed during compilation and do **not** need to be distributed.
