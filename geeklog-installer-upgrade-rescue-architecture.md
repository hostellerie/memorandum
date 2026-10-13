# Geeklog Install / Upgrade / Rescue and legacy recovery architecture

> **Status: architecture proposal (October 2026).** Planned features are not claims of completed code. Read alongside the [2.1.1 → 2.2.2 multisite field guide](geeklog-2.1.1-to-2.2.2-multisite-upgrade-lessons.md) and [runtime policy](geeklog-runtime-support-policy.md).

## Purpose

Help site owners move forward after years without updating Geeklog, including installations stranded on obsolete PHP. Keep recovery from an old SQL dump distinct from upgrading an existing Core database in place. Geeklog **2.2.2 is the destination baseline**; old Geeklog/PHP runtimes are accepted as historical source inputs, not new support targets.

## Responsibility boundaries

| Component | Owns | Does not own |
| --- | --- | --- |
| **Monitor** | Site health, diagnostics, alerts and recommendations (read-only by default) | Core schema upgrades or SQL imports |
| **Migrator** | Staging SQL dumps, source recognition, adapters for legacy Geeklog/glFusion/WordPress, fresh-site data import, ID mapping, verification reports | Updating an installed Core or replacing an occupied destination |
| **Native Geeklog Installer / Upgrade / Rescue** | New Core installation, controlled in-place schema upgrades, preflight, upgrade checkpoints and recovery from partial upgrades | CMS-to-CMS content conversion |

Do **not** create an additional **Updator** plugin. Share minimal documented reporting contracts, not direct access to another component's private tables.

## Choose a path based on the condition of the source

1. **Viable in-place upgrade:** inventory and restore-test backups; run a non-mutating Core preflight; migrate a *staging copy* first; confirm schema/plugin compatibility; then execute a controlled production upgrade.
2. **Legacy, broken or PHP-incompatible installation:** export or recover the SQL dump plus media files; deploy clean Geeklog 2.2.2; use **Migrator** to stage, analyze, dry-run and explicitly import into the empty target; copy referenced files and verify content.
3. **Interrupted Core upgrade:** invoke independent **Rescue** diagnostics; inspect the actual table/column/index state and completed steps. Never infer safe replay just from `database_version`; require a validated repair or a restore from backup.

No recovery tool should need to run insecure old PHP publicly. If old code must be examined, isolate it.

## Mandatory in-place upgrade preflight

- Identify the selected HTTP host and exact site context, database name, table prefix, active `db-config.php`/ `siteconfig.php`, shared files and site-specific paths before modifying anything.
- Detect declared version **and actual schema**, active/removed optional plugins, PHP and SQL capabilities, free space and writable paths.
- Check **table, column and index** preconditions before every SQL step. An absent Polls table, for example, must not trigger queries against it; skip only explicitly optional operations and log them.
- Never silently rewrite shared configuration or another site's persistent data during a selected-site upgrade.
- Require a **verified, restorable backup** of site SQL and critical files, separate from disposable cache.

## Execution and Rescue safeguards

- Preflight and dry run have no write side effects on target content.
- Every mutation requires authorization, CSRF controls when web-driven, and explicit confirmation.
- Keep a persistent, per-site journal of stages attempted, completed and failed; track schema facts and failures, not passwords or sensitive payloads.
- Implement safe retries only when independently verified against the current schema and content. MySQL DDL may commit implicitly: **never promise universal transactional rollback**.
- Preserve old/abandoned plugin data when possible; do not install obsolete plugin code merely to read its tables.
- Rescue must function without the regular admin bootstrap where possible, preferably with protected CLI support. Any web rescue endpoint needs separately designed authentication and hardening.
- Verify row counts, IDs, users and password-reset needs, relationships, paths, URLs and media before marking a job successful.

## Historical SQL recovery (Migrator)

Keep source SQL and file archives untouched. Import only allowed statement forms into private, isolated staging tables; do not execute arbitrary original SQL over live Geeklog Core tables. Detect source signatures and version uncertainty, table prefixes and recoverable types; report unsupported content. Preserve IDs when safe, otherwise maintain explicit mapping. Require a fresh Geeklog 2.2.2 destination; provide dry-run results, explicit import and downloadable verification reports.

A SQL dump does **not** contain physical uploaded images or media. Export a missing-file manifest and copying instructions. Historical schema compatibility is **test-evidenced per version**, never presumed universal.

## Acceptance tests and development order

- **P0:** Protect shared multisite configuration, recognize missing optional tables, detect schema/version mismatch, test partial failure and retry, verify backup checkpoints.
- **P1:** Add independent diagnostic/Rescue mode and clear per-stage status while keeping maintenance authorization explicit.
- **P1:** Expand Migrator's legacy schema inventory and SQL-only recovery reports; verify user/content counts and missing assets.
- **P2:** Improve CLI operation and progressive compatibility fixtures.

Automated isolated tests should cover clean installs, known historic upgrades (including 2.1.1 → 2.2.2), absent Polls, partial databases, separate site databases sharing Core files, PHP 5.6.40 and PHP 8.1/8.3 where supported, and malformed SQL/archive inputs. Publish supported source-version combinations only after testing.

## Related repositories

- [Migrator](https://github.com/hostellerie/migrator/tree/develop_1.1.0)
- [Monitor](https://github.com/hostellerie/monitor)
- [Core installer fixes](https://github.com/hostellerie/geeklog-2.2.2/tree/fix/multisite-upgrade-installer)
