# 🪨 Cairn

**A self-hostable GitHub alternative built for agents.** Server-centric version control for monorepos of any size and any file type, from source code to multi-GB game assets. C++23 core, web forge on [Ultimate](https://github.com/developerz-ai/ultimate).

> Status (2026-11): design stage. Docs only, no code yet. "Cairn" is a working title.

## Why
Git puts all history in every clone and merges through queues. With hundreds of agents landing changes, that breaks. Cairn takes the best idea from each version control system ([catalog](docs/idea/11-ideas-from-every-vcs.md)) and builds one model:

- **Server owns history.** Commits, rebases and lands run server-side. Clients fetch only what they need.
- **Land by compare-and-swap.** No serial merge queue. Concurrent lands batch into one ref update per round; verification runs after merge, with flake-aware bisect and auto-revert.
- **Cost scales with the change.** A changeset is a delta against its parent. Root trees, blame, path history and the commit graph are derived indexes.
- **Immutable, content-addressed objects** (BLAKE3, zstd, FastCDC chunks). Any cache tier serves anything.
- **Every file type.** Versioned typemap: locks for unmergeable assets, chunked multi-GB files, previews, purge and archive.
- **Optional working copies.** Sparse checkout, a virtual filesystem (FUSE, NFS/FSKit, ProjFS), or none at all: query the API.
- **Self-hostable.** One box on PostgreSQL + disk; scale to YugabyteDB + S3 + regional caches.

## Architecture
```
cairn CLI ─▶ cairnd (node) ─▶ cairn-cache (regional) ─▶ cairn-server ─▶ PostgreSQL / YugabyteDB
                                                           ▲   └──────▶ disk / S3-compatible
                      cairn-web forge (Ultimate) ──JSON────┘
```

## Docs
| Start here | |
|---|---|
| What and why | [docs/idea/](docs/idea/README.md) |
| How to build it | [docs/implementation/](docs/implementation/README.md) |
| Decisions | [docs/decisions/](docs/decisions/README.md) |
| Roadmap (M0–M5) | [15-roadmap.md](docs/implementation/15-roadmap.md) |
| Agents | [CLAUDE.md](CLAUDE.md) · [.claude/](.claude) (rules, subagents, commands, skills) |

## Built for agents to build
The repo is set up so coding agents, including smaller models, can do great work:
- `CLAUDE.md`: the contract (rules, layout, commands, boundaries).
- `.claude/rules/`: path-scoped rules that load automatically (C++, hot paths, determinism, formats, storage, web, docs).
- `.claude/agents/`: architect, cpp-implementer, reviewer, perf-engineer, dst-hunter, security-auditor, web-forge, docs-keeper.
- `.claude/commands/`: `/planx`, `/feature`, `/check`, `/review`, `/bench`, `/hunt`, `/adr`.
- `.claude/skills/`: dev loop, formats cheat sheet, deterministic simulation, performance patterns.

## Credits
Cairn builds on published work from Meta's **Mononoke** (Bonsai data model), **Sapling** (segmented changelog) and **EdenFS**; Google's **Piper**, **CitC** and **TAP**; **Jujutsu**; **Dolt** (prolly trees); **Perforce** and **Unity Version Control** (typemaps, locking); Epic's **Lore**; and Anthropic's description of **Scion**. It is independent and not affiliated with any of them. Sources: [prior art](docs/idea/02-prior-art.md).
