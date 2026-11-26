# DuckDB 2.0.1 for ImtCore

The libraries and headers here are copied unchanged from DuckDB's official release packages. ImtCore uses only the C API in `duckdb.h`. DuckDB calls it stable and recommends it for applications, while its C++ API is internal and has not shipped in the release packages since 2.0.

## Contents

| Path | Source asset of [v2.0.1](https://github.com/duckdb/duckdb/releases/tag/v2.0.1) |
|---|---|
| `include/*.h` | Any `duckdb-shared-libs-*.tar.gz`. The headers are the same on every platform. |
| `lib/macOS_universal/libduckdb.dylib` (Git LFS) | `duckdb-shared-libs-osx-universal.tar.gz` |
| `lib/GCC_Linux_x64/libduckdb.so` | `duckdb-shared-libs-linux-amd64.tar.gz` |
| `lib/Windows_x64/duckdb.dll`, `duckdb.lib` | `duckdb-shared-libs-windows-amd64.tar.gz` |

- `duckdb_v2.h` and `duckdb_extension_v2.h` declare the C API v2. ImtCore doesn't use it, because DuckDB marks it as not yet stable.
- On Windows, the official DLL serves Debug and Release builds alike. The C API passes no C++ objects across the DLL boundary, so the CRT of the application does not matter.
- `ImtCore/Config/CMake/DuckDb.cmake` links the library and copies `duckdb.dll` next to the binaries.

## Upgrading

1. Download the three `duckdb-shared-libs-*.tar.gz` assets of the new release and copy their files over the paths above.
2. Read the "breaking changes" of the release notes and the migration guide. Pay particular attention to the storage format: files written by a newer DuckDB can't be opened by an older one.
3. Rebuild `imtduckdb` and the applications that use it. Only a changed `duckdb.h` can require code changes in ImtCore.
