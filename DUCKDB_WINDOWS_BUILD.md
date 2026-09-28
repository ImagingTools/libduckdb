# DuckDB on Windows: CRT mismatch with the C++ API

## TL;DR

DuckDB's own CMake builds its DLL with a **static CRT** (`/MT`, `/MTd`). Our applications use the **dynamic CRT** (`/MD`, `/MDd`), which is the Qt default. The DuckDB C++ API passes `std::string`, `std::unique_ptr`, `std::vector` and exceptions across the DLL boundary, so the two sides disagree on heap and STL layout.

Symptoms are crashes and misbehavior that look unrelated to our code. **Fix: build DuckDB ourselves with the dynamic CRT** (one-line patch below) and ship per-config binaries in `3rdParty/libduckdb`.

This applies to **Windows only**. See [Linux / macOS](#linux--macos).

## Symptoms we saw

Using the prebuilt DuckDB C++ package (headers + `duckdb.dll` + `duckdb.lib`) from a Debug build:

1. The `.duckdb` file was never created after `duckdb::DuckDB db(path)`.
2. First-chance exception in `duckdb!duckdb::Transaction::SetReadWrite` (noise, but a hint that something was off).
3. `0xC0000005` access violation when reading a query result or error. The address was ASCII text read as a pointer (`0xa0a5d30203a` = `": 0]\n\n"`), which points to struct layout disagreement between headers and binary.

After rebuilding DuckDB from source, the file was created, but then:

4. Debug assertion, on destruction of a `std::string` that crossed the boundary:

   ```
   File: minkernel\crts\ucrt\src\appcrt\heap\debug_heap.cpp
   Line: 996
   Expression: __acrt_first_block == header
   ```

   Call stack ended in `xmemory` (`std::allocator::deallocate`) called from the `std::string` destructor.

## Root cause

`dumpbin /dependents duckdb.dll` on our self-built Debug DLL:

```
WS2_32.dll
RstrtMgr.DLL
KERNEL32.dll
```

There is no `MSVCP140D.dll`, `VCRUNTIME140D.dll` or `ucrtbased.dll`, so the CRT is linked statically into the DLL and it has its own heap. The application uses the shared debug CRT. A `std::string` allocated in one module and freed in the other hits the debug heap check.

DuckDB's top-level `CMakeLists.txt` (v1.5.5, line 62) sets this explicitly, which is why passing `-DCMAKE_MSVC_RUNTIME_LIBRARY=...` on the command line has no effect:

```cmake
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
```

The compile flags in `build.ninja` confirm `-MTd`.

## Fix

### 1. Patch DuckDB and build it

In DuckDB's top-level `CMakeLists.txt`, add the `DLL` suffix:

```cmake
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
```

Then:

1. Delete the DuckDB build directory (a clean reconfigure is required).
2. Configure and build with the **same compiler and toolchain as the application** (for us: MSVC 2022, x64).
3. Build **Debug and Release separately**. Both configs are affected by line 62.
4. Verify:

   ```
   dumpbin /dependents duckdb.dll
   ```

   Debug must list `MSVCP140D.dll`, `VCRUNTIME140D.dll`, `ucrtbased.dll`. Release must list the non-`D` versions.

### 2. Copy artifacts into the repo

From the DuckDB build's `src` directory:

- `duckdb.dll`
- `duckdb.lib` (the **import library**, roughly 2 MB)

Do not use `duckdb_static.lib` (over 1 GB in Debug, and it needs all the third-party sub-libraries).

Copy the **headers from the same source tree** (`src/include/*`) into `3rdParty/libduckdb/include`. Headers and binary must come from the exact same version.

### 3. CMake

Link `duckdb.lib` and copy `duckdb.dll` next to the binary. **Do not** define `DUCKDB_STATIC_BUILD`.

The DLL must be present at runtime (exe directory or `PATH`). Linking the import library only resolves symbols at link time. Without the DLL the app fails to start (`0xC0000135`).

### 4. clangd false positive (editor only)

clangd reports:

```
attribute 'dllimport' cannot be applied to a deleted function
```

Clang rejects `dllimport` on DuckDB classes with deleted members. MSVC accepts it, so the real build is fine. Silence it in the editor only, with a `.clangd` file:

```yaml
CompileFlags:
  Add: [-DDUCKDB_STATIC_BUILD]
```

Do not add `DUCKDB_STATIC_BUILD` to the CMake target while linking the DLL. It removes `dllimport` and breaks linking.

## Linux / macOS

Not expected to be affected. The problem is specific to Windows, where each module can carry its own CRT and heap. On Linux and macOS, the C++ standard library and `malloc` are shared process-wide.

Things to check (not tested yet):

- **Linux:** the `.so` and the application need the same `_GLIBCXX_USE_CXX11_ABI` setting (the default on modern GCC). Don't build the app with libstdc++ debug modes such as `_GLIBCXX_DEBUG`.
- **macOS:** both sides use libc++.
- **Runtime lookup:** ensure the `.so`/`.dylib` is found at runtime (rpath or copied next to the binary).
- Header version must match the binary exactly on every platform.

## Checklist for a new DuckDB upgrade

1. Update the source tree and re-apply the line 62 patch (check that the line still exists and has not moved).
2. Clean build Debug and Release.
3. `dumpbin /dependents duckdb.dll` shows the matching dynamic CRT for each config.
4. Copy DLL, import lib and the **matching** headers.
5. Remove any old `duckdb.dll` from exe directories and `PATH`. Confirm the loaded path in the debugger's Modules window.

## Alternatives

- **C API (`duckdb.h`):** plain C, so CRT, config and compiler do not matter, and the official prebuilt DLL works as is. It requires rewriting the C++ wrapper layer (`CDuckSqlDriver`, `CDuckSqlResult`) against the C API.