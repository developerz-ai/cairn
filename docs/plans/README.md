# 🗂️ Plans

Created by `/planx` (architect agent). One folder per plan:

```
docs/plans/YYYY/MM/DD/1NN-<slug>/     # 1NN starts at 101 each day; slug ≤ 5 words
├── overview.md        # goal, success check, invariants touched, slice list
├── 01-<aspect>.md     # one slice: self-contained, < 1 day
├── 02-<aspect>.md
└── status.md          # the only tracker
```

## Slice template
```md
# 01 — <what this slice delivers>
Goal: <one sentence>
Read: <doc paths, path:line refs>
Change: <files to create/edit>
Steps:
1. <step> — verify: `<command>` → <expected output>
Done when: `bin/check` green + <specific test names>
Out of scope: <what not to touch>
```

## status.md
```md
| Slice | State | Owner | PR | Notes |
|---|---|---|---|---|
| 01 | todo / doing / review / done | agent-name | #12 | |
```

Rules: reference, don't paste code. Every step has a `verify:`. No adjective without a check.
