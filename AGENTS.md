# AGENTS.md

## Project shape

- Pi extension package that replaces or augments Pi's TUI with Claude Code-style rendering, compact assistant rounds, rich diffs, mouse interaction, context/session references, and shell/footer features.
- `extensions/index.ts` is the package entrypoint. It wires shell features, renderer layers, and optional features from the singleton `config`.
- `extensions/renderer/index.ts` owns `/ccstyle` and the runtime lifecycle. `default-mode.ts` handles normal tool cards; `compact-mode.ts` handles compact round summaries; `tool/`, `mouse/`, and `transcript-refresh.ts` contain their shared machinery.
- `extensions/config/config.ts` owns config loading, normalization, defaults, migration, and persistence. Runtime writes must use `updateConfig`; the active object is intentionally mutated in place.
- `extensions/utils/patch-keys.ts` is the source of truth for cross-module patch keys and ownership. All cross-reload keys must remain `Symbol.for(...)`; do not replace them with module-local `Symbol()` values.

## Commands

- Requires Node.js `>=22.19.0`; install the locked dependency graph with `npm ci`.
- Focused test: `node --test tests/<name>.test.ts`.
- Full verification: `npm run lint && npm run typecheck && npm test && npm run format:check`.
- Renderer changes also require `npm run docs:tool-render`; commit both generated files under `docs/tool-render-examples-*.md`.
- `npm run typecheck` covers only `extensions/**/*.ts` and `tests/**/*.ts`; execute `npm run docs:tool-render` when changing its TypeScript generator because that script is outside the TypeScript project.
- Manual TUI check: `pi -e .`. On Windows, `test.bat [pi args...]` temporarily swaps an installed npm copy for this checkout and restores the previous package configuration on exit.
- Release validation order is encoded by `preversion`: format, lint, typecheck, tests, then renderer-doc generation. Do not publish by bypassing it.

## Known baseline test failures

- With the current locked install (`@shikijs/cli` resolves to 4.4.1), `node --test tests/tool-diff.test.ts` reproducibly fails only these assertions: `write collapsed preview uses writeDiffCollapsedLines independently of edit` and `writeDiffCollapsedLines 0 shows stats only until expanded`.
- The rendered content is present, but Shiki inserts ANSI SGR sequences between source tokens, so plain regexes such as `/const value0 = 0/` cannot match the raw output. The likely test fix is to apply the already-imported `stripVTControlCharacters` before plain-text assertions.
- Until fixed, do not skip `tests/tool-diff.test.ts` wholesale. Run the full suite and tolerate only these exact two failures with the same ANSI-fragmented assertion output; any additional failure is a regression.

## Change constraints

- Prototype/global patches must be safe across `/reload`, resume, compaction, stale shutdowns, and concurrent headless runtimes. Preserve ownership-guarded teardown through `PatchRegistry`; run `tests/patch-registry.test.ts` and `tests/runtime-patch-isolation.test.ts` after lifecycle changes.
- TUI patches are installed only for `ctx.mode === "tui" && ctx.hasUI`. Do not let print/headless sessions replace or tear down the main TUI's patches.
- The `write` tool override is deliberately registered during `session_start`, after other extensions have loaded, so it can detect and yield to another owner. Do not move it to module initialization.
- Reload/resume can leave components created from older module instances. Prefer structural compatibility checks and transcript refresh paths over assuming `instanceof` or current prototypes are sufficient.
- Renderer behavior is coupled to width, ANSI preservation, fullscreen/regular TUI ownership, expansion state, and mouse hit-testing. Add or update the nearest focused test rather than validating only string output manually.
- When adding or renaming a config option, grep `Config`, `DEFAULT_CONFIG`, `normalizeConfig`, `formatConfigStatus`, `config/panel.ts`, both READMEs, and related tests so persistence, UI, docs, and defaults stay aligned.

## Navigation symbols

- Runtime assembly: `ensureTuiInstallation`, `applyStyleMode`, `refreshCurrentTranscript`.
- Patch lifecycle: `PatchRegistry`, `patchRegistry`, `GLOBAL_TOOL_RENDER_PATCH`, `COMPACT_MODE_PATCH_KEY`.
- Renderer ownership: `installDefaultMode`, `installCompactMode`, `installToolGrouping`, `installMessageDisplayRendering`.
- Transcript recovery: `refreshMountedTranscript`, `refreshTranscriptComponent`.
- Tool rendering: `toolCallSummary`, `ExpandedToolIoView`, `renderRichToolResult`, `WriteExecutionMetadataStore`.
- Compact behavior: `buildMessageSummary`, `installCompactThinking`, `markCompactRoundToolExpanded`.

## Maintenance

- Keep progress and next-task notes in `LOG.md`, `WORKFLOW.md`, or the repository's future equivalent; never put transient progress in this file.
- After structural edits, refresh the navigation-symbol list above. Add a pitfall only when repository evidence shows agents are likely to repeat it; remove guidance that is no longer true.
