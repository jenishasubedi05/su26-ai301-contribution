# Contribution #1: consider giving more context for safe schema change based on usage

**Contribution Number:** 1
**Student:** Jenisha Subedi
**Issue:** https://github.com/graphql-hive/console/issues/6892
**Status:** Phase IV COMPLETE

---

## Why I Chose This Issue

I chose this issue because it sits at the intersection of API design and frontend UI, which matches my skills in both backend and frontend development. The problem is clearly scoped — surfacing client inclusion context in the schema check view — making it a great first contribution to a real-world GraphQL tooling platform.

I'm also excited to learn more about how GraphQL schema validation and client inclusion lists work in practice. Contributing to graphql-hive will give me hands-on experience with a production-grade open source tool used by real engineering teams, which is exactly the kind of work I want on my resume.

---

## Understanding the Issue

### Problem Description

When a schema change is marked as "safe" because of the client inclusion list, the schema version/schema check view does not explain why it is considered safe. Developers are left without context about what made the change safe.

### Expected Behavior

The schema version/schema check view should display information about which clients are in the inclusion list that caused the change to be classified as safe, giving developers full visibility into the safety determination.

### Current Behavior

The view shows the change is safe but provides no explanation or context about the client inclusion list that determined it.

### Affected Components

- Schema version/schema check view (dashboard frontend)
- Client inclusion list API logic

---

## Reproduction Process

### Environment Setup
Setting up the local development environment for graphql-hive/console on Windows surfaced several real cross-platform issues, which I diagnosed and fixed:

1. **Postgres port conflict**: A pre-existing native PostgreSQL 17 service on Windows was bound to port 5432, intercepting connections meant for the Docker Postgres container and causing "password authentication failed for user postgres" errors during `pnpm local:setup`. Fixed by stopping the native `postgresql-x64-17` Windows service.

2. **ClickHouse bind-mount permission error**: ClickHouse failed with `filesystem_error: in rename: Permission denied` when writing to its data directory, because the compose file bind-mounted `./.hive-dev/clickhouse/db` from the Windows filesystem into the container. Atomic file renames don't work reliably across the WSL2/Windows filesystem boundary. Fixed by switching ClickHouse's data volume to a named Docker volume (`clickhouse_data`) instead of a bind mount.

3. **Windows path/glob resolution bugs in build tooling**: `pnpm build` failed for `@hive/app` because `tsup` could not resolve `src/server/index.ts` as an entry point, despite the file existing. The root cause is that `tsup`'s internal glob matching treats Windows backslash path separators as glob escape characters, so absolute Windows paths (e.g. `C:\Users\...\index.ts`) don't match as literal entry points. The same issue affected `vite.config.ts` and `packages/web/app/src/server/index.ts`, both of which used `new URL('.', import.meta.url).pathname` to compute `__dirname` — on Windows this produces a malformed path with a leading slash before the drive letter (e.g. `/C:/Users/...`), which Rollup/tsup cannot resolve. The same pattern also affected `packages/libraries/external-composition/build-example.mjs`. I fixed all of these by (a) replacing `.pathname` with Node's `fileURLToPath()` for correct `__dirname` resolution in ESM on Windows, and (b) converting resolved entry paths to forward slashes via `.split(sep).join('/')`.

4. **Unix-style inline env vars in npm scripts**: Several scripts (e.g. `graphql:generate`: `VERBOSE=1 graphql-codegen ...`) use the `VAR=value command` syntax, which only works in POSIX shells and throws `'VERBOSE' is not recognized...` in PowerShell. I removed the inline env var from `graphql:generate` since it only controlled log verbosity.

After these fixes, `pnpm i`, `pnpm local:setup`, `pnpm generate`, and `pnpm build` all completed successfully across all 22 packages, and `pnpm dev:hive` successfully started the full local stack.




### Steps to Reproduce


1. Run `pnpm dev:hive` to start all Hive services locally
2. On first boot, `@hive/server` (GraphQL API, port 3001) and `@hive/commerce` both attempt to run `graphile-worker` schema migrations concurrently against the same Postgres database, and both crash with `duplicate key value violates unique constraint "migrations_pkey"` / `"pg_namespace_nspname_index"`
3. On retry, the schema/migration tables already exist from the first attempt, and `@hive/server` and `@hive/app` start successfully — `@hive/app` (dashboard) listens on `http://localhost:3000` and `@hive/server` (API) on `http://localhost:3001`
4. Registered a local account at `http://localhost:3000`, created an organization (`jenisha`) and a single-schema project (`test-project`) with a `development` target
5. In the target's Settings → Breaking Changes tab, configured Conditional Breaking Changes: 0% traffic threshold, 1 total operation, over 30 days, with usage data sourced from the `development` target itself — this is the exact mechanism issue #6892 refers to

### Reproduction Evidence

- **My findings**: The Settings → Breaking Changes UI confirms that Hive supports marking a breaking change as "safe" based on real usage data (the "Conditional Breaking Changes" feature). However, nothing in this settings page or in the schema check results view explains, after the fact, *which* usage condition or excluded client caused a specific change to be downgraded from breaking to safe — which is precisely the gap issue #6892 describes. A developer reviewing a "safe" schema check has to manually cross-reference the target's Conditional Breaking Changes configuration to understand why a change passed.
- I also encountered and documented a first-boot migration race condition between `@hive/server` and `@hive/commerce` (both running `graphile-worker` migrations concurrently) — a real environment finding worth noting for future Windows contributors, though unrelated to #6892 itself.



---

## Solution Approach
### Analysis

Issue #6892 asks for the schema check view to explain *why* a schema change is considered "safe" when that determination is based on the client inclusion list or usage thresholds (i.e., `conditionalBreakingChangeConfiguration` on the target). Currently, the backend evaluates this condition and downgrades the change's criticality, but does not return any explanatory metadata about *why* to the dashboard.

### Proposed Solution

1. In the schema/policy evaluation logic (in `packages/services/policy` and/or `packages/services/schema`), extend the conditional-breaking-change evaluation so that when a change is downgraded from breaking to safe due to usage/client exclusion, the result includes metadata describing the reason (e.g. which client(s) were excluded, or that the usage was below the configured threshold).
2. Expose this metadata on the `SchemaChange` GraphQL type in the relevant `.graphql` schema definitions, then run `pnpm graphql:generate` to regenerate types across the monorepo.
3. In `packages/web/app`, update the React component(s) that render schema check results to display this context — e.g. a small info icon or inline note next to changes marked "safe": "Safe because affected usage is below the configured threshold" or "Safe because the only affected client(s) are excluded: <client name>".

### Implementation Plan 

**Understand:** A schema change marked "safe" due to conditional breaking change settings (usage threshold or client exclusion) gives no UI explanation of why.

**Match:** The schema check view already renders per-change criticality and human-readable descriptions; the new context should follow this same rendering pattern for consistency.

**Plan:**
1. Locate the conditional-breaking-change evaluation code path (in `packages/services/policy` or `packages/services/schema`) where a change's criticality is downgraded due to usage/client exclusion
2. Add reason metadata to the evaluation result returned by the API
3. Update the relevant `.graphql` schema files to expose this field on `SchemaChange`, then run `pnpm graphql:generate`
4. Update the React component(s) in `packages/web/app/src` that render schema check results to display the new context
5. Add unit tests covering the new metadata field and its UI rendering

**Implement:** Branch created and pushed: https://github.com/jenishasubedi05/console/tree/feature/6892-safe-change-context

**Review:** Will follow CONTRIBUTING.md style guide; run `pnpm lint` and type-checks before pushing implementation commits.

**Evaluate:** Manually verify in the local dashboard that the new context appears on a schema change downgraded to safe via Conditional Breaking Changes; add an automated test for the new metadata field.



---

## Testing Strategy
Manually verified the code changes compile without errors after running `pnpm graphql:generate`. Next steps: add unit tests for the new `criticalityReason` field and verify the UI displays the reason correctly in the local dashboard.



---

## Implementation Notes
## Changes Made

1. **`packages/services/api/src/modules/schema/providers/inspector.ts`**
   Added `reason: change.criticality.reason` to the `HiveSchemaChangeModel.parse()` call so the criticality reason is passed through the model.

2. **`packages/web/app/src/components/target/history/errors-and-changes.tsx`**
   - Added `criticalityReason` to both `ChangesBlock_SchemaChangeWithUsageFragment` and `ChangesBlock_SchemaChangeFragment` GraphQL fragments
   - Added UI display: when a change is safe based on usage and has a `criticalityReason`, it now renders inline next to "Safe based on usage data"

### Branch
https://github.com/jenishasubedi05/console/tree/feature/6892-safe-change-context



---

## Pull Request

**PR Link:** https://github.com/graphql-hive/console/pull/8182

**PR Description:**
This PR surfaces the existing `criticalityReason` field through to the frontend UI. When a schema change is marked safe based on usage data, the criticality reason now displays inline next to "Safe based on usage data" in the schema check results view.

**Status:** Awaiting review
No feedback received yet — awaiting review.

## Learnings & Reflections

[To be filled upon completion]

---

## Resources Used

- https://github.com/graphql-hive/console/issues/6892
- https://github.com/graphql-hive/console/blob/main/docs/CONTRIBUTING.md
