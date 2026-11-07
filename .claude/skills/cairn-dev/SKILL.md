---
name: cairn-dev
description: How to build, test, run and debug Cairn locally. Use when starting work, when a build/test fails, or when unsure which command to run.
---
# Cairn dev loop

| Task | Command |
|---|---|
| First time | `bin/setup` |
| Build | `bin/build` (preset `dev`); `bin/build asan` for sanitizers |
| One module's tests | `bin/test object` |
| One test | `bin/test object -tc="tree roundtrip*"` |
| Full gate | `bin/check` (≡ CI) |
| Run server + web | `bin/dev all` → server :7420, web :3000 |
| Sim | `bin/sim --seed 42`, `bin/sim --seeds 1000 --json` |
| Fuzz | `bin/fuzz tree_decode -max_total_time=60` |
| Bench | `bin/bench checkout` |

## When something fails
1. Read the **first** error only. Fix it. Rebuild.
2. Link error → missing target dep in `CMakeLists.txt` (check layering doc before adding).
3. Sanitizer report → top frame in `src/` is the bug; fix there.
4. Test flaky → it's a bug (time, order, or shared state). Never retry-until-green.
5. Same error twice → stop, re-read the owning doc, then ask.

## Where things are
`CLAUDE.md` → layout table. Docs: `docs/implementation/README.md`.
