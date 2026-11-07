---
paths: ["src/**/*.{cpp,hpp,h}", "tests/**/*.{cpp,hpp}", "bench/**/*.cpp", "tools/**/*.cpp", "**/CMakeLists.txt", "cmake/**"]
---
# C++ rules

- C++23, `CXX_EXTENSIONS OFF`. No C++20 modules.
- Errors: `std::expected<T, cairn::Error>` + `[[nodiscard]]`; propagate with `CAIRN_TRY`. Never `throw`; catch third-party throws at the adapter.
- Invariants: `CAIRN_ASSERT(cond, "msg")` — aborts in all builds.
- `switch` on enums: no `default:`.
- Ownership: `unique_ptr` default; `shared_ptr` needs a justification comment. No `new`/`delete`/`malloc`. `std::jthread` only, owned.
- Params: `const T&` read · `T&` mutate · `T` sink · `span`/`string_view` views · `T*` non-owning nullable.
- Strong ids (`ObjectId`, `ChangesetId`, `ChangeId`, `RefName`) — never raw bytes/strings across APIs.
- Naming: `snake_case` fns/vars, `PascalCase` types, `kConst`, `member_`, `cairn::<module>`.
- `#pragma once`. ≤ 500 LOC per file, ≤ 60 lines per function.
- Heavy headers (grpc, libpq, aws, fuse) only in adapter `.cpp` files.
- New dependency → decision record first.
- Test in the same change: `tests/unit/<module>/<file>_test.cpp` (doctest).
- Before done: `bin/fmt`, `bin/test <module>`, then `bin/check`.
- Full doc: `docs/implementation/08-cpp-standards.md`.
