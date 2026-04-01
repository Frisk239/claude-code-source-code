# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **decompiled TypeScript source** extracted from the npm package `@anthropic-ai/claude-code` v2.1.88. The published package ships a single ~12MB bundled `cli.js`; this repo contains the **unbundled source** (~1,900 files, ~30K+ LOC) extracted from the tarball for research purposes. It is **not** a fork or modified version.

The source is **incomplete** — 108 feature-gated modules referenced by `feature()` branches are missing (dead-code-eliminated in the original Bun build). These cannot be recovered.

## Build Commands

```bash
npm run prepare-src          # Patch source: replace Bun intrinsics, inject MACRO values
npm run build                # Full build: prepare-src + esbuild bundle → dist/cli.js
npm run check                # TypeScript type checking (tsc --noEmit)
npm run start                # Run built CLI (node dist/cli.js)
```

**Requirements**: Node.js >= 18, npm >= 9.

**Build caveat**: The original code uses Bun compile-time intrinsics (`feature()`, `MACRO`, `bun:bundle`). The build scripts replace these with runtime stubs. `feature('X')` is replaced with `false` — all feature-gated code is eliminated. ~108 modules are missing and need manual stubs. See `QUICKSTART.md` for troubleshooting.

No test runner or linter is configured.

## Architecture

### Core Agent Loop

```
cli.tsx → main.tsx → REPL.tsx (interactive) or QueryEngine.ts (headless/SDK)
                       ↓
              query.ts: while-true loop
              → Call Claude API → check stop_reason → execute tools → append results → repeat
```

### Key Modules

| Module | Purpose |
|--------|---------|
| `src/query.ts` | Core agent loop — the main orchestration logic |
| `src/QueryEngine.ts` | Headless/SDK query lifecycle, `submitMessage()` → `AsyncGenerator<SDKMessage>` |
| `src/Tool.ts` | Tool interface: `call()`, `validate()`, `checkPermissions()`, `render()`, `prompt()` |
| `src/tools.ts` | Tool registry, presets, filtering |
| `src/main.tsx` | REPL bootstrap (large file, ~4,700 lines) |
| `src/commands.ts` | Slash command definitions |
| `src/state/AppState.tsx` | Zustand-like global state store with React context |

### Directory Layout

- `src/entrypoints/` — CLI, SDK, MCP entry points
- `src/tools/` — 40+ tool implementations (Bash, FileRead/Edit/Write, Glob, Grep, Agent, MCP, etc.)
- `src/commands/` — 80+ slash commands
- `src/components/` — React/Ink terminal UI components
- `src/hooks/` — ~70 React hooks
- `src/services/` — Business logic: API client, analytics, MCP protocol, compact, plugins
- `src/bridge/` — Claude Desktop / remote bridge
- `src/ink/` — Forked Ink terminal rendering framework
- `src/constants/` — Prompts, OAuth, tool limits, XML tags
- `src/coordinator/` — Multi-agent coordinator mode
- `src/skills/` — Skill loading system
- `src/plugins/` — Plugin system (bundled + loader)
- `src/vim/` — Vim-style input motions/operators
- `src/utils/` — Utility modules (~40+ subdirectories)
- `stubs/` — Build stubs for Bun intrinsics (`bun-bundle.ts`, `macros.ts`, `global.d.ts`)
- `scripts/` — Build scripts (`build.mjs`, `prepare-src.mjs`, `transform.mjs`)
- `docs/` — Analysis reports (EN/JA/KO/ZH)

### Technology Stack

TypeScript/TSX with React + forked Ink for terminal UI. Original build uses Bun with compile-time intrinsics; this repo uses esbuild with stubs. Key dependencies: `@anthropic-ai/sdk`, `@commander-js/extra-typings`, Zod, lodash-es, Chalk. AsyncGenerator pattern used throughout for streaming.

## Build System Details

The build pipeline (`scripts/build.mjs`):
1. Copy `src/` → `build-src/`
2. Transform: `feature('X')` → `false`, `MACRO.VERSION` → `'2.1.88'`, `bun:bundle` imports → stubs
3. Create entry wrapper injecting MACRO globals
4. Bundle with esbuild, iteratively creating stubs for missing modules

`tsconfig.json` uses `"strict": false`, targets ES2022 with bundler module resolution, and maps `bun:bundle` to `stubs/bun-bundle.ts` via path aliases.

## Deep Analysis Reports

The `docs/` directory contains quadrilingual analysis reports covering: telemetry & privacy, hidden features & model codenames, undercover mode, remote control & killswitches, and future roadmap. See `README.md` for full index.
