---
paths: ["docs/**", "README.md", "CLAUDE.md", ".claude/**"]
---
# Docs rules

- Lead with the rule. Fragments over sentences. Tables for 3+ rows. Emoji in H1 only.
- One topic per file. One owning doc per fact; link, don't copy.
- Date load-bearing claims (`As of 2026-11`). Cite primary sources.
- New doc → add a line to its folder `README.md`.
- Decisions: `docs/decisions/NNN-slug.md`, never renumber, supersede don't edit.
- Plans: `docs/plans/YYYY/MM/DD/1NN-slug/` (see `docs/plans/README.md`).
- No secrets, private hosts or client names.
