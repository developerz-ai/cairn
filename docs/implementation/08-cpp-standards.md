# 📏 C++ standards

> C++23. Clang + libc++ primary, GCC 14+ must build. Errors are values. One owner per resource. Every rule below has a guard.

## Toolchain
| Item | Choice |
|---|---|
| Standard | C++23, `CXX_EXTENSIONS OFF`. C++26 features only behind a feature-test macro |
| Compilers | Clang 20+ (primary, release builds), GCC 14+ (CI build + tests) |
| Stdlib | libc++ with hardening (`_LIBCPP_HARDENING_MODE_FAST` in release, `_EXTENSIVE` in debug) |
| Linker | lld (mold allowed locally) |
| Modules | No C++20 modules until CMake + both compilers are boring with them |

## Warnings
`-Wall -Wextra -Wpedantic -Wshadow -Wconversion -Wsign-conversion -Wold-style-cast -Wnon-virtual-dtor -Wimplicit-fallthrough -Wswitch -Wnull-dereference -Wundef -Wcast-align`. `-Werror` in `ci` preset, our targets only, never deps.

## Errors
- Expected failures: `std::expected<T, cairn::Error>`, `[[nodiscard]]`. `Error` = `ErrorCode` enum (stable, maps to wire codes) + context string.
- `CAIRN_TRY(expr)` is the only error-propagation macro.
- Invariant violations: `CAIRN_ASSERT` aborts in **all** builds with file:line and message.
- **No exceptions** in our code. Third-party throws are caught at the adapter boundary and converted. Build with `-fno-exceptions` where deps allow (core, object, land).
- `switch` over enums: no `default:`; `-Wswitch` catches missing cases.
- Never swallow an error: log + return, or propagate.

## Ownership
- `std::unique_ptr` by default. `shared_ptr` needs a one-line justification comment.
- No raw `new`/`delete`/`malloc`/`free`. C handles wrapped in move-only RAII types (`cairn::Handle<T, Deleter>`, `ScopeExit`).
- No detached threads. Every thread has an owner that joins it (`std::jthread` + stop token).
- Parameters: `const T&` read, `T&` mutate, `T` sink, `std::span`/`std::string_view` views. Raw `T*` = non-owning, nullable.
- Never hold a pointer into a container across an `co_await` or a lock release; store the id/index.

## Types
- Strong ids: `ObjectId`, `ChangesetId`, `ChangeId`, `RefName` are distinct types (no implicit conversion).
- Units in types: `Bytes`, `Millis`. No naked `int` sizes; `std::size_t` / `std::uint64_t` explicit.
- `enum class` only. `constexpr`/`consteval` for tables (varint limits, mode bits).

## Style
- `snake_case` functions/vars, `PascalCase` types, `kConstant`, members end `_`, namespaces `cairn::<module>`.
- `#pragma once`. Files ≤ 500 LOC, one job each. Functions ≤ 60 lines.
- clang-format (LLVM base, 4 indent, 110 cols). clang-tidy: `bugprone-*`, `performance-*`, `cppcoreguidelines-{owning-memory,no-malloc,init-variables,slicing}`, `modernize-use-{override,nullptr}`, `readability-identifier-naming`.
- No macros except `CAIRN_ASSERT*`, `CAIRN_TRY`, `CAIRN_TRACE*`, and platform `#if`.

## Determinism (core, object, derive, land, merge)
- No wall clock: inject `Clock&`. No `std::random_device`: inject `Rng&` (named PCG streams).
- No iteration over `unordered_*` where output order matters (encoding, hashing, replies).
- No thread identity in logic. Guard: `scripts/lint/determinism.sh`.

## Compile speed
Unity builds in CI/release, shared PCH, pimpl at subsystem boundaries, heavy headers only in adapter `.cpp`. Budget: one-file rebuild + link < 5 s. `bin/build-profile` runs `-ftime-trace`.

Performance rules: [09-cpp-performance.md](09-cpp-performance.md). Memory rules: [10-memory.md](10-memory.md).
