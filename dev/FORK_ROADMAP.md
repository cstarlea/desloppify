# desloppify fork roadmap

_Built from a multi-agent review on 2026-10-05._

**Scope:** fork `cstarlea/desloppify` at `3a7735d5`, which is the same commit as upstream `peteromallet/desloppify` main and v1.0. The review covered 8 areas and about 130 raw findings. A skeptic pass removed the refuted ones; the rest are grouped below into about 75 deduplicated items. Statuses are taken from the skeptic verdicts: **confirmed**, **partially-confirmed** (some details wrong, called out where it matters) and **unverifiable**.

---

## 1. Executive summary

**State of the project.** The tool runs, is fast, and is used heavily: 3,161 stars, 256 forks, 62 open issues and 124 open PRs. Upstream has stopped. The last community merge was #450 on 2026-03-19, there have been no maintainer comments since v1.0, and contributors are closing their own PRs with "repo appears unmaintained" (#711, #718, #720). The community is splitting into competing forks: awdemos, citizenadam's auto-merge fork, randomparity/mending and MercurialUroboros. Upstream main CI has been red since April, but PyPI publishing doesn't depend on CI, so 1.0 shipped with failing tests.

**TypeScript in particular.** The TS plugin is about 14k lines. It is mostly per-line regex (165 non-test regex call sites), and it uses tree-sitter only for the cohesion phase, even though a `tsx` grammar spec is already bundled (`_framework/treesitter/specs/scripting.py:229`).

On four real repos (ky, zod, vercel/commerce, trpc), roughly **65–75% of sampled issues were false positives**. Test-health scores came out backwards: ky, which is well tested, scored 3.2%, and commerce, which has no tests, scored 7.6%. Most errors trace back to a few shared pieces of plumbing:
- a regex import graph whose tsconfig reader is wrong (JSONC, `references`, `extends`, baseUrl and workspaces all fail);
- comments and strings are never stripped before matching;
- a global `matches[:50]` cap that drops most smells;
- tsc and knip integrations that **fail silently while coverage still reports "full", confidence 1.0**.

### Biggest risks, in order

1. **The auto-fixers corrupt or silently change user code.** Every TS fixer, and `move`, produced broken syntax or changed behaviour on ordinary code. The worst examples:
   - the debug-logs fixer deletes `const result = await saveUser(user)` (critical);
   - empty-if-chain deletes a non-empty `else { doC(); }`;
   - unused-vars deletes the wrong declaration;
   - unused-params renames destructured props to `_subtitle`, which breaks types.

   Nothing checks the output before it is written, and the dry-run "before → after" view only shows "before".
2. **Silent zeros look like clean code.** If `typescript` isn't installed locally, `npx tsc` downloads the unrelated `tsc` npm package ("This is not the tsc command you are looking for") and the tool reports 0 unused symbols. Knip never runs with the default `--path src`. An offline tree-sitter grammar download fails and is logged at debug level only. In every case the score goes **up**.
3. **State and plan integrity.**
   - All languages share one `plan.json`, so scanning a second language permanently wipes the first language's skips, clusters and queue.
   - The strict score can never recover from real fixes.
   - One invalid issue makes `load_state` throw away the entire state, and the `.bak` rotation then overwrites the last good copy.
4. **Agent guidance points the wrong way.** The #1 queue item on ky is a fake "shallow tests" issue. Orphan false positives come with "delete dead files" advice, including for Next 16's required `proxy.ts`. The deprecated detector calls a published library's public types "safe to delete".

### Biggest opportunities

- **One resolver fixes about six detectors.** A real tsconfig/workspace resolver, combined with AST-based import extraction, fixes orphaned, single_use, cycles, facade, coupling and test_coverage together.
- **Most of the fixes already exist as PRs.** I verified that #744 (27 reviewed PRs) + #617 + #760 + #629 merge onto main with no conflicts and give **7072 passed, 0 failed**. MercurialUroboros has 28 commits (about 17 of them TS false-positive fixes) that fast-forward cleanly.
- **The extension points are good enough.** FrameworkSpec, ToolIntegration, `make_tool_phase(cwd_fn=…)`, PARSERS and `dimensions.override.json` are enough to add tsc type errors, ESLint/Biome/oxlint, and more frameworks without new architecture.
- **Positioning.** Users want a maintained successor. Shipping first, under a clear name, while keeping authorship, wins the community back from the scattered forks.

**Decision for week 1:** put the TS code-rewriting fixers behind an explicit `--unsafe` flag until the validation gate (§2 #1) lands, and make the tsc/knip failure modes report reduced coverage instead of zero.

---

## 2. Fix first: critical and high correctness problems

Order is by user harm. Each item lists file references and key evidence, with the effort in parentheses.

### A. Fixers that break or silently change code

1. **Validation gate for all fixers and `move`** (M). `fixer_io.py:67-77` writes as soon as the content differs. `preview.py:25-48` dry-run never renders the new content. `move/apply.py` finishes with "Done.".
   - Fix: parse before and after (tree-sitter tsx/ts, or `ts.createSourceFile` diagnostics), refuse to write when the parse-error count rises, and show a real unified diff in dry-run. Optionally add `--verify` to run `tsc --noEmit` and auto-revert.
   - Caveat (skeptic): a parse gate won't catch semantic deletions, because items 2, 3, 4 and 10 produce output that parses. Those need their own fixes as well.
2. **Debug-logs fixer deletes side-effecting declarations** (critical, M). `logs_cleanup.py:101-143` (`find_dead_log_variables`) removes any `const X = …` whose name only appears in a removed log, for example `const result = await saveUser(user)`. On multi-line declarations it removes only the first line, which leaves `a: 1,\n};`. Remove this cascade, or limit it to literal initializers with no calls or awaits.
3. **Logs fixer removes whole physical lines** (high, M). `fixers/logs.py:38-53`. It deletes `a(); console.log(..); b();` entirely, turns a brace-less `if` into a guard for the next statement, deletes exported arrow declarations, and breaks JSX handlers and ternaries. Only remove standalone ExpressionStatements, using AST ranges.
4. **`remove_empty_blocks` runs over the whole file** (high, S). `fixers/logs.py:58`, `logs_cleanup.py:73-226`. It deletes deliberate `.catch(() => {})` handlers, which turns them into unhandled rejections, and deletes `const unsubscribe = subscribe(() => {`. Restrict it to blocks that contained a removed line.
5. **The logs detector and `detect logs --fix`** (high/medium, S). `detectors/logs.py:31,107-123`. The detector matches inside template strings and comments, and treats `${user.id}` as a tag. `_fix_logs` removes only the first line of a multi-line call. Delete `_fix_logs` and route through a hardened `fix_debug_logs`.
6. **unused-imports statement collector** (high, M). `import_rewrite.py:139,239-250`. After a side-effect import or `import x = require()` in no-semicolon code, it emits `import, { b } from 'lib';`. "(entire import)" deletes `import 'reflect-metadata'`. Treat side-effect and `require` imports as complete statements, and never delete side-effect imports.
7. **Import rewriter drops bindings when a comment contains `}`** (high, M). `import_rewrite.py:60`. Its alias handling (`x as y, y as z`) deletes `z`, which is still used. It also reformats imports by adding `;` and expanding them to multiple lines. Fix by deleting only the specifier token range.
8. **unused-vars removes the wrong declaration** (high, M). `vars.py:24-44`. It never checks that the declarator name matches the reported name. Results: `const double = n => 2` is deleted when `n` is the unused one, `let a=1,b=2` loses `b`, write-only `let count` is deleted, and `const prev = i++` is deleted.
9. **Destructuring rewrite splits on raw commas and braces** (high, M). `vars.py:180-214`. It produces `const { 2), b } = obj;`, changes the string `'}'` to `' }'`, and reports names as removed when they were not.
10. **empty-if-chain deletes a non-empty `else`** (high, S). `if_chain.py:41-68` together with `detector_flow.py`. It deletes `else { doC(); }` and leaves an orphan `else doIt();`. Only fix chains that start at a real `if` and where every branch is empty.
11. **unused-params renames destructured props** (high, S). `params.py:61-73`. `{ title, subtitle }` becomes `{ title, _subtitle }` (TS2339). The fallback detector's `line.find` produces `c_onst`. unused-params is fed by the `vars` detector (`_fixers.py:91`), so it also renames locals.
12. **`move` breaks TS layouts** (high, L). `move.py:97-165`, `move/apply.py:49-58`, `move/cmd.py:59`.
    - Sibling imports inside a moved directory are rewritten against the old location (`'../../feature/y'`).
    - `.js` (NodeNext) specifiers are never matched.
    - Importers outside `src/` are skipped.
    - Chained `str.replace` runs in set order, so replacements depend on hash seed and can double-rewrite; it also rewrites string literals.

    Until this is fixed, abort when a moved file has importers that can't be rewritten.

### B. Silent false negatives that inflate the score

13. **tsc invocation** (high, S–M). `detectors/unused.py:31-36,53-55,87-94`.
    - `npx tsc` runs without `--no-install` or `stdin=DEVNULL`, so it can fetch the bogus `tsc` package and return 0.
    - It hard-codes `extends: ./tsconfig.app.json`.
    - The return code is never checked.
    - It writes `tsconfig.desloppify.json` into the user's repo.
    - It parses only TS6133/6192 and drops TS6196/6198/6205 (3 of 7 real findings reported in the fixture).

    Partially-confirmed nuance: with the `tsconfig.app.json` hard-code alone, tsc emits TS5083 and **keeps running with default options**. Detection doesn't go to zero, but the user's `paths`, `jsx` and `include` are ignored. The true silent zero is the missing-local-typescript case.

    Fix: resolve `node_modules/.bin/tsc` by walking up the tree, then `npx --no-install`. Use `tsc -p <real tsconfig> --noEmit --noUnusedLocals --noUnusedParameters` (or `-b` for references). Parse all TS61xx codes. Record reduced coverage on failure.
14. **Knip adapter** (high, S–M). `knip_adapter.py:34,45,50,107-126`, `exports.py:17-20`.
    - It checks `<scan path>/node_modules/.bin/knip` and runs from the scan path, so the default `src` never runs it.
    - It uses `pos` (a character offset) as the line number.
    - It drops `files`, dependencies, `enumMembers` and duplicates.
    - It passes `--no-gitignore`.
    - The potential equals the failure count.

    Fix: run from the nearest package root (`--workspace` in monorepos), read `line`/`col`, and record "skipped" rather than 0.
15. **Smell detector truncates to 50 matches globally** (high, S). `smells/__init__.py:119`. zod has 734 `as any` but only 50 survive, all in `packages/bench`, so the library core is invisible. Remove the cap from the data path.
16. **Leaf files never enter the graph** (medium, S). `deps/__init__.py:69-73`. Dead constant and type modules are never reported as orphaned, and the graph-size denominator is understated. Seed the graph with every TS file.

### C. False positives that come with harmful advice

17. **tsconfig resolution** (high, M). `deps/resolve.py:58-92`. It uses strict `json.loads` on JSONC, never follows `references` (the Vite template layout), uses `del base_dir` so extended paths get the wrong base, only follows a single string `extends` (skipping `@repo/…` and arrays), uses only `targets[0]`, and ignores baseUrl-only imports. Every failure falls back to `{'@/':'src/'}`. On commerce this produced 16 out of 16 false orphans plus the top action "delete dead files". `test_coverage.py:162-163` separately hard-codes `@/` and `~/`.
18. **Test-quality recognises only Jest/Chai** (high, M). `test_coverage.py:22-54`. It misses AVA `t.is`, `expectTypeOf`, `expect.soft`, backtick names and `.each`. ky's retry.ts (446 assertions) is reported as "0 assertions" and is the #1 queue item.
19. **The test-health score is inverted** (high, M). `_issue_gaps.py` (transitive_only at full weight), `discovery.py:15,65`. Treat coverage through a tested public entry as covered, and fix the "production files" and "checks" labels.
20. **async_no_await body extraction** (high, S). `detector_core.py:99-121` takes the first `{`, which can be destructured params or a type literal. This hits every Next route handler. **Cherry-pick #629.** The concise-arrow case is still open.
21. **Next.js 16** (high/medium, S–M).
    - `next lint` was removed, so `specs/nextjs.py:547` silently reports nothing. **Cherry-pick #760.**
    - `proxy.ts` is flagged as an orphan and becomes the top action, because `orphaned.py:39-43` lacks proxy, forbidden, unauthorized and manifest.

### D. Core engine and state

22. **Shared `plan.json` across language states** (high, M). `_plan/persistence.py:197-199`, `plan_reconcile.py:358`, `scan_issue_reconcile.py:219`. Reproduced: a Python scan supersedes the TS skips and clusters permanently and rebuilds the Python state from the TS plan. Fix with per-language `plan-{lang}.json` plus a migration.
23. **The strict score penalises real fixes** (high, S). `policy/core.py:194-198`, `merge_issues.py:174-195`. `auto_resolved` now means "scan-confirmed fixed" (since 573c1973) but still counts as a failure. Manual "fixed" claims raise strict, and `verified_strict` never clears. Reproduced: a fully fixed codebase is stuck at 97.5. The docs are inconsistent: scoring.md documents this, README and SKILL.md say wontfix only.
24. **One invalid issue discards the whole state** (medium, but total and silent loss, S). `_state/persistence.py:236-257,310`. There is no `.corrupted` rename, `.bak` gets overwritten, and `version=None` raises a TypeError. This is high-risk for a fork that will add new statuses (version skew). Quarantine the bad issues and keep the rest.
25. **Persisted exclusions are applied after state-path and language resolution** (medium, S). `cli.py:162-164`. This writes to the wrong `state-<lang>.json` (#785, #727).

**Critical item outside the TS plugin:** the shared tree-sitter import graph (`_framework/treesitter/imports/graph.py:74-83`) compares absolute resolved paths against a set of relative ones, so **JS and every generic plugin using `make_ts_dep_builder` (Java, Ruby, PHP, Kotlin, Swift) gets zero edges**. It also ignores `require()`. Reported impact: 193 false orphans in #715, and 1090 of 1372 files on a Strapi app in #750. Fix with #750, which needs a hand merge in `graph.py` after #744.

---

## 3. TypeScript improvement roadmap

Overlapping findings are merged; IDs refer to the appendix. Effort scale: S < 1 day, M a few days, L 1–2 weeks, XL multi-week.

### Milestone 0: Stop the bleeding (about 1–2 weeks, mostly S)

Goal: no fixer silently breaks code, no detector silently reports zero, and agents stop getting the worst false positives.

| # | Item | Effort | Rationale |
|---|---|---|---|
| 0.1 | Gate code-rewriting fixers (logs, vars, params, if-chain, imports) behind `--unsafe` until 1.1/2.x land; disable the logs dead-variable cascade | S | Critical semantic deletions (FX-1…FX-11) |
| 0.2 | Apply upstream batch: #617 → #744 → #760 → #629, then hand-resolve #750 | M | Verified green (7072 passed); fixes async body, next lint, the generic graph, Windows runners, RR/Remix entries |
| 0.3 | tsc: resolve a local binary, `--no-install`, `stdin=DEVNULL`, `-p <real tsconfig>` with CLI flags (no temp file in the repo), check rc/TS5xxx, parse 6133/6138/6192/6196/6198/6199/6205 | S–M | TL-1. Removes a network-code-execution risk and silent zeros |
| 0.4 | Knip: run from the package root, read `line`/`col`, drop `--no-gitignore`, report "skipped" | S | TL-2 |
| 0.5 | Add a "reduced coverage" record for tsc, knip, framework tools and tree-sitter init failures; show it in scan output and lower score confidence | S | Fixes "coverage: full, 1.0" while tools fail (TL-1/2, PK-2) |
| 0.6 | Remove `matches[:50]` | S | DT-1 |
| 0.7 | Seed the graph with every TS file | S | GR-5 |
| 0.8 | Make the tsconfig loader JSONC-tolerant and follow `references` (full resolver comes in M1) | S | Biggest single false-positive source (GR-1) |
| 0.9 | Add assertion patterns for AVA/tap/node:test `t.<m>(`, `expectTypeOf`/`assertType`, `expect.soft`, and the broadened `TEST_FUNCTION_RE` | S | DT-12. Removes ky's #1 false item |
| 0.10 | Zones: `test-d/`, `*.test-d.ts`, `e2e/`, `cypress/`, `playwright/`, `*.cy.ts`, `*.e2e.ts`, `bench*/`, `examples/`, `*.config.{ts,mts,cts}`, `*.gen.ts`, `*.generated.ts`, `*_pb.ts`, `gql/`, `@generated` headers; classify directory-level issues | S | DT-11 |
| 0.11 | Security regex: `(?<![\w$.])eval\s*\(`; ignore `innerHTML = ''`; skip comment and string spans | S | DT-7 |
| 0.12 | Next 16: add `proxy.ts`, forbidden, unauthorized, manifest and mdx-components to the conventions | S | GR-6. Removes a harmful "delete" suggestion |
| 0.13 | Deprecated detector: JSDoc `@deprecated` only; never print "safe to delete" for exported or barrel-reachable symbols | S | DT-5 |
| 0.14 | `_categorize_unused`: default to `vars` and add a `params` category; never default to `imports` | S | FX-15 |
| 0.15 | Fix the write path: preserve newline style, BOM, mode and symlinks; decode as UTF-8 | S | FX-14, FX-18 |

### Milestone 1: A trustworthy graph and resolver (about 3–5 weeks)

Goal: one module-resolution layer that matches TypeScript's, shared by deps, orphaned, single_use, cycles, facade, coupling, test_coverage and `move`.

| # | Item | Effort | Rationale |
|---|---|---|---|
| 1.1 | `languages/_framework/node/tsconfig.py`: JSONC parsing; full `extends` chain (string, array and package specifiers via node_modules); `references`; baseUrl relative to the defining config; all `paths` targets; package.json `imports` (#subpath). Consider `tsc --showConfig` when tsc is available. Emit a coverage warning instead of the `@/` fallback | M | GR-1 (merges 5 findings across 4 areas) |
| 1.2 | Workspace map from `workspaces` / `pnpm-workspace.yaml` (name → dir plus exports/main/types/bin); nearest tsconfig per file | L | GR-2: monorepos (zod, trpc, turbo) are currently blind; package entry barrels get flagged as facades |
| 1.3 | Import extraction on the bundled tree-sitter tsx grammar: import/export-from, `export * as`, `import()`, `require`, `import x = require`, `vi.mock`/`jest.mock`, triple-slash refs. Comments and strings are skipped for free. Record `type_only` per edge | M | GR-3, GR-4. Removes false cycles (ky's 16-file "cycle" is type-only) |
| 1.4 | Resolution candidates: `.js→.ts/.tsx`, `.jsx→.tsx`, `.mjs→.mts`, `.cjs→.cts`, directory package.json; add `.mts`/`.cts` to `TS_EXTENSIONS`; one TS source finder that excludes `.d.ts`/`.d.mts`/`.d.cts`; include JS when `allowJs` is set | M | GR-3, GR-7 |
| 1.5 | Entry points from package.json (main/module/exports/bin/scripts, mapping dist→src via rootDir/outDir); add `entry_conventions` to FrameworkSpec and move the `_NEXTJS_*` constants out of `engine/detectors/orphaned.py`; detect frameworks per package | M | GR-6, GR-2 |
| 1.6 | Dynamic imports become real resolved edges instead of the unanchored `endswith` suffix match (`orphaned.py:118-123`) | S | GR-6 |
| 1.7 | Lower orphan confidence when the file has unresolved non-relative specifiers | S | Safety net while the resolver matures |
| 1.8 | `test_coverage` and `move` use the same resolver; basename test mapping restricted to the same package | M | DT-12, FX-12 |
| 1.9 | `--path` sets the project root and state dir, with a test that scanning from another cwd gives identical IDs | M | GR-9 |
| 1.10 | Consolidate the dual graph builders (#615): TS uses the shared tree-sitter builder with the TS resolver plugged in, then rebase #614 (Astro/Svelte/Vue) | L | GR-8. Partially-confirmed: only one builder runs for TS today, but the divergence is real |

### Milestone 2: AST-based detectors and fixers (about 4–8 weeks, can overlap M1)

Prerequisite: make tree-sitter plus a tested tslp range a hard requirement when TS is detected, with a grammar-download preflight (see PK-1/PK-2).

| # | Item | Effort | Rationale |
|---|---|---|---|
| 2.1 | Shared typed TS AST helper: parse once per file via the parse cache; function, class, import and JSX queries; regex kept as fallback | L | AR-2 (ts-regex-architecture) |
| 2.2 | Port the function extractor (async, default, methods, multi-line params) and `_extract_function_body`, including concise arrows | M | DT-2, DT-3 |
| 2.3 | Type-safety smells as queries: `non_null_expression`, nested `as_expression` (`as unknown as`), `any`, `@ts-ignore` in block comments, separate `ts_expect_error_undocumented`; report per-file density | M | DT-9 (partially-confirmed: `ts_nocheck` already exists) |
| 2.4 | Props detector: count properties, include extends/generics/intersections, word-boundary names (fixes `BankStatementRow` being matched as "State") | M | DT-4 |
| 2.5 | Facade: every top-level statement is a re-export (directives and comments allowed); exempt package entries | S | DT-6 |
| 2.6 | Deprecated: attach JSDoc to the AST node; count importers from the graph plus same-file uses; skip `.d.ts` | M | DT-5 |
| 2.7 | Strip comments before every remaining line-regex detector (security, smells, logs) | M | DT-7 (comments-not-stripped) |
| 2.8 | Rewrite the fixers on AST ranges: import specifier deletion that preserves formatting; declarator-matched var removal; destructuring patterns; standalone-statement-only log removal; `name: _name` for params; scoped if-chain removal. Then remove the `--unsafe` gate | L | FX-1…FX-11 |
| 2.9 | Fixer state bookkeeping: fixers return the exact issue IDs they fixed | S | FX-16 |

### Milestone 3: New capabilities (ongoing, prioritised)

| # | Item | Effort | Rationale |
|---|---|---|---|
| 3.1 | **tsc `type_error` detector**: run once, share output with unused (runtime cache), stable IDs `TS####::line`, scored under type safety | M | TL-3. Type errors are currently invisible (TS2322 scores 93%+) |
| 3.2 | **ESLint/Biome/oxlint integration** chosen by config presence (ToolIntegration on a node-tooling FrameworkSpec). `parse_eslint` keeps `ruleId` and severity; add biome and oxlint parsers; tier the type-aware typescript-eslint rules | M | TL-4 |
| 3.3 | **`tsconfig_health` detector**: strict off, missing noUncheckedIndexedAccess / noImplicitOverride / verbatimModuleSyntax, per-package drift; feed strictness into the review context | S | DT-13. Cheap and high value |
| 3.4 | Knip extended categories: unused files (cross-checked with orphaned), unused or unlisted dependencies, unresolved imports, enum members, duplicates | M | TL-2 |
| 3.5 | Replace the original author's residue with config and presets: pattern families, boundary rules, data-layer identifiers. Ship generic defaults (react-query vs SWR vs fetch-in-effect, redux/zustand/jotai). Move Supabase checks into a `supabase` FrameworkSpec. Generalise public-env-secret prefixes (`NEXT_PUBLIC_`, `VITE_`, `PUBLIC_`, `EXPO_PUBLIC_`). Exclude non-applicable detectors from potentials | M | DT-10 |
| 3.6 | Framework specs in user-base order: React Router v7/Remix, SvelteKit, Nuxt, Astro, NestJS, Express/Hono/Fastify, Angular. Each gets deps detection, `entry_conventions` and 2–4 high-signal scanners. Port the MercurialUroboros Nuxt work | XL | DT-14 |
| 3.7 | SFC `<script lang="ts">` extraction for `.vue`/`.svelte`/`.astro` as virtual files with line offsets | M | DT-14 |
| 3.8 | Backend security: unauthenticated server actions and route handlers, Prisma/Drizzle/Kysely raw-unsafe APIs with interpolation, `child_process` with template literals, data-flow-aware secrets | M | DT-15 |
| 3.9 | Class god rules (methods, constructor-injected deps, decorators) chosen per file by content | M | DT-17 (partially-confirmed) |
| 3.10 | TS review overrides for type_safety, dependency_health and test_strategy; split REVIEW_GUIDANCE into react and node sections | S | DT-16 (partially-confirmed) |
| 3.11 | Ingest real coverage (`coverage-final.json`, `lcov.info`) when present | M | DT-12 |
| 3.12 | Project-config detector and domain **disable** that excludes them from scoring rather than suppressing (thresholds already exist in CONFIG_SCHEMA) | M | CE-15 (partially-confirmed) |

---

## 4. Engineering foundation

### 4.1 Tests

- **Golden TS fixture projects (E1, M).** The repo has no `.ts` files at all today. Add projects for vite-react (with references), next-app (Next 16, `proxy.ts`), node-lib (plain tsconfig, exports and bin), a pnpm/turbo monorepo (aliases, `@repo/tsconfig` extends, workspace imports), plus deno and js-only. `test_ts_golden.py` should snapshot sorted (id, detector, file, line, tier) plus potentials, support `--update-golden`, and run under both the core and `[full]` installs. Delete or wire up the dead `tests/snapshots/cli_smoke`.
- **Node CI job (E2, M).** setup-node, `npm ci` with pinned typescript and knip, `pytest -m node`. Real tsc and knip paths: the TS5083 config error, the missing-typescript fallback, real knip JSON.
- **Fixer round-trip property tests (FX-19, M).** For each fixer, the output must parse, a second run must be a no-op, CRLF/BOM/mode must be preserved, and `tsc --noEmit` must not report new errors. Seed this with adversarial inputs like the ones in §2A. Add `move` tests for NodeNext, aliases and directory moves.
- **Isolation (E5, S).** Add an autouse fixture that enters `runtime_scope(project_root=tmp_path)`, plus a guard that fails the run if `<repo>/.desloppify` is touched. Tests currently write into the real repo's `query.json` and `progression.jsonl`.
- **Get a green baseline (E4, S).** Use `sorted(glob)` in `review_commands_cases.py:876` (it fails on tmpfs `/tmp` on Arch/CachyOS), which is what #617 does. Delete the duplicate `tests/review/integration/*` wrappers (179 tests run twice). Fix `testpaths` so it picks up the ruby and r tests and the roslyn stub (76 uncollected tests). Extend the layout guard.
- **Replace refactor tripwires (E9, M).** Swap `inspect.getsource`, `callable(fn)` and file-absence asserts for behaviour tests. `find_replacements` currently has no behaviour test at all.
- Consolidate the three TS test trees into `languages/typescript/tests/{detectors,fixers,golden,node}` (E13, S).

### 4.2 CI and tooling

- Python matrix 3.11–3.14 for tests-core and tests-full (E7); add the 3.14 classifier. Add a tree-sitter matrix for the lower bound and the latest version.
- `ruff check --fix` (496 autofixable, mostly I001), then enforce the configured `E,F,I,B,UP` plus `ruff format --check`. Add mypy over `languages/typescript` and `_framework` (about 163–169 errors, ratchet with per-module ignores). Add import-linter contracts that already hold (languages ↛ app, engine ↛ app, plugin independence) plus a ratcheted base-layer contract (E6).
- A `dev` extra with pinned pytest, ruff, mypy, import-linter, pytest-xdist and pytest-timeout. Check targets should not `pip install` (E11). Add dependabot. Relax the action-version contract test (`startswith`). Add `--timeout` (E12, timings unverifiable, but the risk of a hung subprocess is real).
- Add a Windows job, at least core, to cover the runner UTF-8 issue (#808) and the ESLint `2>/dev/null` / `npx.CMD` issue (#714).

### 4.3 Packaging and dependencies

- **tree-sitter range (PK-1, M).** The floor `tree-sitter>=0.21` crashes, because `QueryCursor` needs ≥0.25. The cap `<1.8` keeps users on 1.6.2. Port the MercurialUroboros dual-generation shim, move to `tree-sitter>=0.25`, `tree-sitter-language-pack>=1.6.2,!=1.6.3,<2` (or test ≥1.15), and replace the exact-string test with a matrix job. Do not merge #611.
- **Grammar download (PK-2, M).** tslp 1.x downloads grammars at runtime, and offline runs silently produce nothing. Add a `desloppify setup --grammars` preflight, make `is_available()` actually try loading the grammar, and report failure as reduced coverage.
- **package-data (PK-3, S).** The wheel is missing the elixir, php and r `review_data/*.json` files. Use one glob, and extend package-smoke to compare the tracked runtime files against the wheel's contents.
- Single-source the version through `importlib.metadata`, and test that `SKILL_VERSION` matches the marker in `docs/SKILL.md`. Fix the release checklist paths and the BSD-only `sed -i ''`.

### 4.4 Core engine

- Per-language plan files plus a migration (CE-1). Rework strict scoring so scan-confirmed resolutions don't count as failures, and add a test that full remediation reaches 100 in every mode (CE-2).
- Make state loading resilient: per-issue quarantine, rename to `.corrupted`, no `.bak` rotation after a failed load, coerce the version field, fsync, preserve the file mode (CE-3).
- Use the existing locks: put every state and plan read-modify-write under `state_lock`/`plan_lock` (partially-confirmed: `plan_lock` has exactly one caller). Trim progression atomically (CE-4).
- Auto-resolve deferred, triaged_out and wontfix issues when their file or detector confirms the issue is gone (CE-5).
- Rewrite `docs/scoring.md` from the code (placeholder dimensions scored 0, all 20 dimensions, `verified_strict`, the status-to-mode table) and fix QUEUE_LIFECYCLE.md (CE-6).
- First-run UX: show the objective score as the headline (marked provisional) until subjective dimensions have been assessed. Make `--profile ci` print plain output with a threshold exit code. Move `cycles` out of the Security dimension (CE-12, partially-confirmed).

### 4.5 Architecture simplification

- **Declarative detector spec (AR-1, L).** One dataclass covering DetectorMeta, zone policy, the file-based flag, the phase callable and the detect-command callable, registered by the plugin. Move the TS, Next.js and Rust entries out of `base/registry/catalog_entries.py`, `engine/policy/zones_data.py` and `_scoring/policy/core.py`. Add `dev scaffold-detector`. This turns a 6–8-file change into 2 files plus a test.
- Rewrite `languages/README.md` from `policy.py` and the real tree: it references `compat/`, `analysis.py` and `phases.py`, none of which exist (AR-3). Delete the 15 dead tree-sitter `_*.py` shims, the dead `help` branch, and the user-facing `dev test-hermes` (AR-4).
- **Scope decision.** Plan, triage and review machinery is about 51k lines (app/commands/plan 18.5k, review 12.9k, engine/_plan 10.6k, intelligence/review 9.1k) against about 8.4k for the TS plugin. Freeze it, collapse the state facades, replace the 84 `work_items`/`issues` fallbacks with a single accessor, put runners behind one adapter, and spend effort on detection accuracy (CE-8, low).

### 4.6 Fork-specific concerns

- **PyPI name.** The fork cannot publish as `desloppify`. Pick a distribution name (for example `desloppify-ng`) but **keep the `desloppify` import package and console script** so users can migrate with a one-line install change. Separately, ask the upstream owner about transferring the name; he is active on other repos.
- **Publishing.** `python-publish.yml` triggers on push to main and isn't gated on CI. Partially-confirmed: it only actually publishes when the version changes. Switch to `release: published` / tag `v*`, verify the tag matches the version, gate on CI, and update `test_ci_contracts.py:95` and `dev/ci_plan.md` together. Disable `tweet-release.yml`, which needs upstream secrets.
- **Upstream pointers.**
  - `update_skill/cmd.py:24` downloads SKILL.md from `peteromallet/main`, which is version skew. Install from bundled package data by default instead.
  - `docs/SKILL.md:259,290,294` points agents at upstream for clone, issues and `uvx` install.
  - `pyproject [project.urls]` and the release checklist also point upstream.

  Template or retarget all of these, and add a maintainer entry to the authors.
- **Attribution.** Keep the upstream LICENSE and copyright notices, and add a NOTICE/CREDITS section naming upstream and the cherry-picked contributors. Merge community PRs with `--no-ff` (as #744 does) so authorship survives. Credit fork-sourced commits (MercurialUroboros, awdemos) in the changelog.
- **Cherry-pick plan.**
  1. #617, for a green baseline.
  2. #744, as the 27-PR integration branch.
  3. #760 (Next 16 lint).
  4. #629 (async body).
  5. #750 (generic graph and `require`), hand-merged in `graph.py`; close #634, #696 and #781 as superseded.
  6. #718 (Windows UTF-8) and #704 (Codex token footer).
  7. MercurialUroboros commits one by one.

  Skip #721 (awdemos found an `unused.py` regression in it) and #611. Use awdemos's per-PR review comments for triage, and consider inviting awdemos to co-maintain so the forks don't fragment further.
- **Community.** Enable Actions for outside-contributor PRs, set required checks on main, and announce the fork on the busiest upstream issues (#501, #615, #705, #665, #715).

---

## 5. Appendix: deduplicated finding table

Severity is the highest adjusted severity among merged items. Verdict "conf" means all merged items are confirmed. "partial" means at least one merged item is partially-confirmed, with details in the main text. "unverif" means unverifiable.

### Fixers and move (area: ts-fixers-safety, plus ts-detector-accuracy and ts-empirical where merged)

| ID | Source IDs | Sev | Kind | Effort | Verdict | Title | Files |
|---|---|---|---|---|---|---|---|
| FX-1 | logs-dead-var-side-effects; logs-template-literal-and-fixer (fixer part) | critical | bug | M | conf | Logs fixer deletes side-effecting declarations and the first line of multi-line declarations | fixers/logs_cleanup.py, fixers/logs.py |
| FX-2 | logs-line-granular-removal | high | bug | M | conf | Line-granular log removal deletes neighbours, changes control flow, breaks JSX and ternaries | fixers/logs.py, fixers/syntax_scan.py |
| FX-3 | logs-filewide-empty-block-removal | high | bug | S | conf | File-wide empty-block removal deletes `.catch(()=>{})` and declarations | fixers/logs_cleanup.py |
| FX-4 | legacy-detect-logs-fix; logs-template-literal-and-fixer (_fix_logs and detector) | high | bug | S | conf | `detect logs --fix` removes the first line only; tag regex matches strings, comments and `${}` | detectors/logs.py |
| FX-5 | imports-statement-collector | high | bug | M | conf | Import collector swallows the next import and emits `import, { b }`; deletes side-effect imports | fixers/import_rewrite.py |
| FX-6 | imports-comment-brace | high | bug | M | conf | `}` or comma in a comment drops bindings; alias handling deletes used names; reformats | fixers/import_rewrite.py |
| FX-7 | vars-direct-removal-wrong-target | high | bug | M | conf | unused-vars removes the wrong declaration or declarators, and write-only vars | fixers/vars.py |
| FX-8 | vars-destructuring-naive-split | high | bug | M | conf | Destructuring split breaks syntax, changes strings, over-reports | fixers/vars.py |
| FX-9 | params-destructured-prefix | high | bug | S | conf | `_`-prefix renames destructured props; `c_onst` corruption; params fed by vars | fixers/params.py, detectors/unused_fallback.py, _fixers.py |
| FX-10 | if-chain-deletes-else | high | bug | S | conf | empty-if-chain deletes a non-empty else; orphan else | fixers/if_chain.py, smells/detector_flow.py |
| FX-11 | no-output-validation | high | robustness | M | conf | No parse gate before writing; dry-run shows no "after" | fixers/fixer_io.py, autofix/preview.py, move/apply.py |
| FX-12 | move-directory-and-specifiers | high | bug | L | conf | move breaks sibling imports, `.js` specifiers, importers outside src | typescript/move.py, move/planning.py, move/cmd.py |
| FX-13 | move-chained-replace | medium | bug | S | conf | Chained global `str.replace` in set order double-rewrites and touches strings | move/apply.py, typescript/move.py |
| FX-14 | write-path-metadata-loss | medium | bug | S | conf | CRLF→LF, symlink replaced, mode 0600, encoding mismatch | fixers/fixer_io.py, base/discovery/file_paths.py |
| FX-15 | unused-categorization-misroutes; unused-miscategorized | medium | bug | S | partial | `_categorize_unused` defaults to "imports"; params labelled "Unused imports" T1 | detectors/unused.py |
| FX-16 | autofix-state-id-mismatch | low | bug | S | partial | Autofix never resolves unused issues (ID format), over-resolves grouped issues | autofix/apply_retro.py, issue_factories.py |
| FX-17 | useeffect-removes-unrelated-comment | low | bug | S | conf | dead-useeffect deletes the preceding `//` line | fixers/useeffect.py |
| FX-18 | bom-first-line-skipped | low | robustness | S | conf | BOM hides the line-1 import (fails safe) | fixers/import_rewrite.py |
| FX-19 | fixer-tests-happy-path-only; ts-untested-smells-and-fixer-roundtrip | medium | test-gap | M | partial | No adversarial or round-trip fixer tests; some smells untested | typescript/tests/test_ts_fixers*.py, smells/catalog.py |

### External tools (TS)

| ID | Source IDs | Sev | Kind | Effort | Verdict | Title | Files |
|---|---|---|---|---|---|---|---|
| TL-1 | tsc-unused-silent-zero; tsc-unused-wrong-config-and-codes; unused-tsc-invocation-fragile; tsc-knip-silent-degradation (tsc); ts-unused-tsconfig-app-hardcode | high | bug/FN | M | partial (app.json "silent 0" did not reproduce; user config ignored instead) | tsc: bogus `npx tsc`, hard-coded tsconfig.app.json, unchecked rc, temp file in repo, drops TS6196/6198/6205 | detectors/unused.py |
| TL-2 | knip-never-runs-default-path; knip-line-is-byte-offset; knip-adapter-broken; tsc-knip-silent-degradation (knip) | high | bug | M | conf | Knip runs in the wrong dir, uses pos as line, drops categories, fake potential, silent when missing | detectors/knip_adapter.py, detectors/exports.py |
| TL-3 | tsc-type-errors-discarded | high | missing-feature | M | conf | tsc type errors never reported | detectors/unused.py, typescript/__init__.py |
| TL-4 | no-eslint-biome-oxlint-for-ts | high | missing-feature | M | conf | No ESLint/Biome/oxlint for TS; parse_eslint drops ruleId | typescript/__init__.py, generic_parts/parsers.py |
| TL-5 | nextjs16-next-lint-dead | high | bug | S | conf | `next lint` removed in Next 16, so silently zero (fix: #760) | frameworks/specs/nextjs.py |

### Graph, resolver and entry points

| ID | Source IDs | Sev | Kind | Effort | Verdict | Title | Files |
|---|---|---|---|---|---|---|---|
| GR-1 | tsconfig-jsonc-and-references; tsconfig-parse-jsonc-references; tsconfig-paths-parsing; tsconfig-extends-wrong-base; import-resolution-misses-real-configs | high | FP | M | conf | tsconfig: JSONC, references, extends base, package/array extends, baseUrl, targets[0], `@/` fallback | deps/resolve.py, test_coverage.py |
| GR-2 | workspace-package-imports-unresolved; monorepo-workspaces-blind | high | FP/FN | L | conf | Workspaces and package exports not modelled; framework detection not per-package | deps/__init__.py, deps/resolve.py, facade.py, frameworks/detection.py, orphaned.py |
| GR-3 | import-syntax-coverage; mts-cts-ignored; dts-mts-inconsistent-discovery | high | FP | L | partial (.d.ts score impact overstated) | Missing require / import=require / baseUrl / .jsx / triple-slash; .mts/.cts ignored; generated .ts zoned production | deps/__init__.py, plugin_contract.py, detectors/io.py, _zones.py |
| GR-4 | comments-strings-create-edges-typeonly-cycles; type-only-import-cycles | medium | FP | S–M | conf | Comment and string imports create edges; `import type` creates false cycles | deps/__init__.py |
| GR-5 | leaf-files-never-orphaned | medium | FN | S | conf | Files with no imports never enter the graph | deps/__init__.py, engine/detectors/orphaned.py |
| GR-6 | orphan-entrypoints-and-dynamic-matching; entry-points-not-package-aware; extensions-entrypoints-allowjs; nextjs-16-stale-conventions | medium | FP | M | conf | No package.json, config or framework-route entries; loose dynamic match; Next 16 proxy.ts; allowJs | plugin_contract.py, engine/detectors/orphaned.py, nextjs/scanners.py |
| GR-7 | (merged into GR-3 and GR-6: extensions-entrypoints-allowjs .mts part) | — | — | — | — | — | — |
| GR-8 | dual-ts-dep-graph-615 | medium | maintainability | L | partial | Separate regex TS graph vs the shared tree-sitter graph | deps/__init__.py, treesitter/imports/graph.py |
| GR-9 | path-not-project-root | medium | robustness | M | conf | `--path` from another cwd: state in cwd, `../` IDs, different results | deps/resolve.py, discovery.py, unused.py |
| GR-10 | ts-graph-relative-paths-and-cjs | critical | FP | S | conf | Shared tree-sitter graph: zero edges with relative paths; no `require()` (JS and generic plugins) | treesitter/imports/graph.py, specs/scripting.py |

### Detectors (TS)

| ID | Source IDs | Sev | Kind | Effort | Verdict | Title | Files |
|---|---|---|---|---|---|---|---|
| DT-1 | smell-global-50-truncation | high | bug | S | conf | Global `matches[:50]` drops most smells | smells/__init__.py |
| DT-2 | ts-function-body-extractor; async-no-await-body-extraction | high | FP | S | conf | Body extractor grabs param or return-type braces (async_no_await FPs) | smells/detector_core.py, detector_flow.py |
| DT-3 | function-extractor-regex | low | FN | M | conf | Function extractor misses async/default/methods, truncates | extractors_functions.py |
| DT-4 | props-bloat-regex | medium | FP | M | conf | Props detector counts lines, skips extends, generics, intersections | detectors/props.py |
| DT-5 | deprecated-detector-fp-fn; deprecated-detector-unsafe-advice | medium | FP | M | conf | Any "deprecated" text matches; misses async/default/method; word-grep importers; "safe to delete" on public API | detectors/deprecated.py, base/signal_patterns.py |
| DT-6 | facade-syntax-coverage | low | FN | S | conf | Facade misses multi-line, JSDoc, `export * as`, `export type *`, `'use client'` | detectors/facade.py |
| DT-7 | security-eval-innerhtml-fp; comments-not-stripped; security-keyword-heuristics | medium | FP | S–M | conf | eval matches `page.$eval`/`redis.eval`/comments; JSDoc examples become secrets; keyword heuristics | security/patterns.py, security/detector.py, smells |
| DT-9 | ts-smell-regex-gaps | medium | FN | M | partial (ts_nocheck already exists) | Non-null, block @ts-ignore, double-cast gaps; ts-expect-error lumped with ts-ignore | smells/catalog.py |
| DT-10 | author-specific-heuristics; project-specific-residue | medium | maintainability | M | conf | useToolSettings families, supabase, shared→tools boundary, VITE-only secrets | patterns/catalog.py, concerns.py, phases_coupling.py, security/patterns.py, review.py |
| DT-11 | zone-misclassification (+ zone parts of dts-mts and security-eval) | medium | FP | S | partial (mechanism misdescribed) | test-d, bench, e2e, config, generated not zoned; dir issues always production | _zones.py, engine/policy/zones.py |
| DT-12 | assertion-recognition-jest-only; test-health-score-inverted; test-coverage-heuristics-narrow; basename-test-mapping-cross-package | high | FP/bug | M | conf | Jest-only assertions; inverted test-health; narrow idioms; cross-package basename mapping | test_coverage.py, engine/detectors/test_coverage/* |
| DT-13 | tsconfig-strictness-ignored | medium | missing-feature | S | conf | tsconfig strictness flags never read | deps/resolve.py |
| DT-14 | only-nextjs-framework | high | missing-feature | XL | conf | No framework support beyond Next.js; SFCs unanalysed | frameworks/registry.py, plugin_contract.py |
| DT-15 | security-node-backend-gaps | medium | missing-feature | M | conf | No server-action auth, raw-SQL or child_process checks | security/file_checks.py, line_checks.py |
| DT-16 | ts-review-prompts-python-centric | low | missing-feature | S | partial | type_safety prompt is Python-flavoured; TS holistic list is thin | review_data/dimensions*.json, review.py |
| DT-17 | god-rules-react-only | low | FN | M | partial | God rules are React-only; no class-level rules | phases_config.py, extractors_components.py |

### Core engine, CLI and config

| ID | Source IDs | Sev | Kind | Effort | Verdict | Title | Files |
|---|---|---|---|---|---|---|---|
| CE-1 | shared-plan-across-lang-states | high | bug | M | conf | One plan.json across languages wipes the other language's skips and clusters | _plan/persistence.py, plan_reconcile.py, scan_issue_reconcile.py |
| CE-2 | strict-penalizes-real-fixes; strict-score-auto-resolved | high | bug | S | partial (strict rule documented in scoring.md) | Strict never recovers from real fixes; manual claims raise it | _scoring/policy/core.py, merge_issues.py |
| CE-3 | invariant-failure-discards-state | medium | robustness | S | conf | One bad issue loses the whole state; .bak overwritten | _state/persistence.py, schema.py |
| CE-4 | locks-unused-lost-updates | medium | bug | M | partial (plan_lock has 1 caller) | Unlocked read-modify-write causes lost updates | _state/persistence.py, _plan/persistence.py |
| CE-5 | deferred-never-reconciled | medium | bug | S | conf | Deferred, triaged_out and wontfix never auto-resolve | merge_issues.py |
| CE-6 | docs-vs-code-scoring | medium | docs | S | conf | scoring.md and QUEUE_LIFECYCLE contradict code | docs/scoring.md, dev/QUEUE_LIFECYCLE.md |
| CE-7 | subjective-gaming-surface | low | robustness | M | partial | Subjective 75% taken as-is; disabled integrity plumbing | state_integration_subjective.py, review/importing/policy.py |
| CE-8 | plan-subsystem-complexity; workflow-machinery-dominates | low | maintainability | L | partial | 10.6k-line plan engine, facades, 84 dual-key lookups, workflow code dominates | engine/_plan, app/commands/plan, state*.py |
| CE-9 | carried-forward-dims-never-expire | low | robustness | S | conf | Carried-forward dimensions never expire | _scoring/state_integration.py |
| CE-10 | concerns-ignore-suppression-scope | low | FP | S | partial | Concerns count suppressed issues | _concerns/state.py |
| CE-11 | recompute-quadratic | low | performance | S | conf | Recompute is O(dims × issues × modes) | _scoring/subjective/core.py |
| CE-12 | score-and-agent-instructions | medium | robustness | M | partial | Headline 15–21/100 from zeroed subjective dims; ci profile carries LLM blocks; cycles under Security | engine/_scoring, app |
| CE-13 | exclusions-after-state-path | medium | bug | S | conf | Exclusions applied after language and state resolution | cli.py |
| CE-14 | lang-detection-walks-to-ancestor | medium | bug | S | conf | Auto-detect walks to an ancestor package.json; Swift scanned as TS | helpers/lang.py, swift/__init__.py |
| CE-15 | config-file-disable-detectors | low | missing-feature | M | partial (config exists; no disable-from-scoring) | No detector or domain disable | base/config, engine/policy/zones.py |
| CE-16 | windows-runner-utf8 | medium | bug | S | conf | Codex runner Popen is locale-dependent on Windows | review/runner_process_impl/attempts.py |
| CE-17 | windows-eslint-npx | medium | bug | S | conf | JS ESLint `2>/dev/null` / npx.CMD on Windows; no `--no-install` | javascript/__init__.py, generic_parts/tool_runner.py |

### Tests, CI, packaging, architecture, upstream

| ID | Source IDs | Sev | Kind | Effort | Verdict | Title | Files |
|---|---|---|---|---|---|---|---|
| E1 | ts-no-fixture-golden-tests | high | test-gap | M | conf | No TS fixtures or e2e scan; snapshots dead | desloppify/tests/fixtures, tests/snapshots/cli_smoke |
| E2 | ts-external-tools-always-mocked | high | test-gap | M | conf | tsc and knip always mocked; no Node in CI | test_ts_unused.py, ci.yml |
| E3 | main-ci-red-publish-ungated; publish-on-push-to-main | high | test-gap/packaging | S | partial (publish only on version change) | CI red on main; PyPI publish ungated, on push | .github/workflows/python-publish.yml, ci.yml, test_ci_contracts.py |
| E4 | flaky-glob-order-test; ci-red-on-main | medium | bug | S | conf | glob-order test fails on tmpfs (fix #617) | tests/review/review_commands_cases.py |
| E5 | tests-leak-state-into-repo | medium | robustness | S | conf | Tests write into the repo's .desloppify | desloppify/conftest.py |
| E6 | lint-gate-subset; ci-gates-shallow; mypy-excludes-ts; importlinter-trivial | low | maintainability | M | partial (cited "real defects" mostly benign) | Lint 4 codes, mypy 16 files, 1 import contract | Makefile, pyproject.toml, .github/importlinter.ini |
| E7 | ci-single-python | low | test-gap | S | conf | CI only on py3.11 | ci.yml |
| E8 | testpaths-miss-ruby-r | medium | test-gap | S | conf | 76 tests never collected | pyproject.toml |
| E9 | coverage-farming-tests | medium | test-gap | M | conf | getsource and callable tests; move rewrite untested | tests/commands/test_direct_coverage_priority_modules.py, test_ts_move.py |
| E10 | duplicate-review-test-wrappers | low | maintainability | S | conf | 179 tests run twice | tests/review/integration/* |
| E11 | ci-infra-hygiene; release-process-drift | low | robustness/docs | S | conf | Make targets reinstall; unpinned tools; checklist wrong paths, BSD sed | Makefile, dev/release/RELEASE_CHECKLIST.md |
| E12 | suite-speed-xdist | low | performance | S | unverif | xdist about 3x; no timeout | pyproject.toml |
| E13 | ts-tests-scattered | low | maintainability | S | conf | TS tests in 3 trees | languages/typescript/tests, tests/lang/typescript |
| PK-1 | treesitter-version-range; tree-sitter-pack-pin | medium | packaging | M | conf | Floor crashes (QueryCursor), cap blocks working releases | pyproject.toml, treesitter/analysis/extractors.py |
| PK-2 | treesitter-runtime-download-silent | medium | robustness | M | conf | Offline grammar download silently drops findings | _framework/treesitter/__init__.py |
| PK-3 | package-data-missing-review-overrides | medium | packaging | S | conf | Wheel omits elixir, php, r review_data JSON | pyproject.toml |
| PK-4 | fork-publishing-identity | medium | packaging | M | partial | Fork inherits PyPI name, upstream update-skill URL, SKILL.md links | python-publish.yml, update_skill/cmd.py, docs/SKILL.md |
| AR-1 | adding-ts-detector-touchpoints | medium | maintainability | L | conf | New TS detector touches 6–8 files incl. base and engine | base/registry/catalog_entries.py, engine/policy/zones_data.py |
| AR-2 | ts-regex-architecture | medium | maintainability | XL | conf | Regex-based TS plugin; tree-sitter optional, only used for cohesion | typescript/__init__.py, extractors_functions.py |
| AR-3 | languages-readme-stale | medium | docs | S | conf | Plugin guide describes nonexistent files | languages/README.md |
| AR-4 | dead-compat-shims | low | maintainability | S | conf | 15 dead tree-sitter shims, dead help branch, test-hermes | _framework/treesitter/_*.py, cli.py |
| UP-1 | maintainer-abandonment-pr-backlog | high | maintainability | L | conf | Upstream dead; 124 PRs; CI never approved | — |
| UP-2 | integration-pr-744-ready | high | maintainability | M | conf | #744 + #617 + #760 + #629 merge clean, 7072 passed | (PR set) |
| UP-3 | mercurial-fork-ts-fixes | medium | FP | M | conf | 28 fast-forwardable commits (~17 TS false-positive fixes) | smells, deps, orphaned.py |

Reproduction fixtures were built in a temporary scratch area during the review and were not kept. Re-create them as part of the golden fixture suite (§4.1).
