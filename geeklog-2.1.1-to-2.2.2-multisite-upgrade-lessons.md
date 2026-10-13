# Geeklog 2.1.1 → 2.2.2: multisite upgrade field guide

> **Historical migration guide, October 2026.** This guide documents an actual migration of 15 Geeklog sites from the Geeklog 2.1.1 era to 2.2.2. It is **not** a requirement to maintain runtime compatibility with Geeklog 2.1.1. The active development baseline is Geeklog 2.2.2 and PHP 5.6.40; see [runtime support policy](geeklog-runtime-support-policy.md).
>
> **Scope:** practical risk controls and failure modes from one shared-files, separate-databases deployment. File paths, SQL statements, version checkpoints, and plugin choices are illustrative, not universal recipes. Do not run destructive SQL or change a stored database version merely because it worked on another site.

## 1. Why this upgrade was difficult

The deployment used **one shared Geeklog Core**, shared plugin and theme files, and **15 distinct site databases** selected according to the HTTP host. Each site's content, configuration, logs, images and persistent plugin data needed to remain isolated.

A shared-file replacement immediately affects all 15 sites, while database and plugin migrations must run **once per database**. This creates an unavoidable transition period: one code version, different database schema versions.

The migration ultimately completed across all 15 sites, but failures exposed assumptions in the installer, upgrade scripts, legacy customizations and deployment process.

## 2. Upgrade checklist: before touching production

1. **Inventory every site:** hostname, database name, table prefix, declared Geeklog version, actual schema, active plugin names and versions, active theme, PHP version, site-specific paths and scheduled jobs. Record the exact mapping without publishing credentials.
2. **Record shared versus local state:** Core, plugins and themes may be shared; site-specific `db-config.php` selection, `siteconfig.php` bootstrap context, databases, media, uploads, cache, logs and persistent plugin data must remain logically isolated.
3. **Back up all databases and all relevant files.** Also capture a restore-tested snapshot of the shared Core, configurations, crontab and persistent data. Keep a separate database export before retrying a partially failed migration.
4. **Plan downtime:** set each site's `site_enabled` flag appropriately and suspend cron jobs, scheduled email sends and background tasks. A single global toggle can accidentally affect every site.
5. **Validate PHP and file permissions.** Geeklog 2.2.2 can run in this deployment with PHP 5.6.40; separately test deployment-specific custom code and plugins. Confirm installer paths and log locations under the *active site's* configuration.
6. **Run a pilot site first.** Review database changes and logs before upgrading the next site. Retain a rollback plan, not only a backup.

Do not assume that a successful installation screen proves every plugin or front-end feature works.

## 3. Declared version can disagree with the actual database schema

During the migration, some databases recorded `gl_vars.database_version = '1.8.0'` although inspection strongly indicated a much newer Geeklog schema. Treat this as a **diagnostic inconsistency**, not an instruction to upgrade from 1.8.0 or permission to edit `gl_vars` immediately.

Useful *read-only indicators* include:

```sql
SELECT name, value FROM gl_vars WHERE name = 'database_version';
SHOW COLUMNS FROM gl_stories LIKE 'text_version';
SHOW COLUMNS FROM gl_blocks LIKE 'cache_time';
SHOW COLUMNS FROM gl_storysubmission LIKE 'text_version';
SHOW COLUMNS FROM gl_blocks LIKE 'device';
SHOW COLUMNS FROM gl_topics LIKE 'tid';
SHOW COLUMNS FROM gl_vars LIKE 'value';
SHOW TABLES LIKE 'gl_topic_assignments';
```

These are **sample indicators**, not a complete schema signature for Geeklog 2.1.1. Verify the database name and table prefix, then compare schema and migration history against the exact Geeklog release scripts. The presence or absence of a few columns alone cannot prove that every intermediate migration completed.

**Never automatically rewrite `database_version`.** If there is convincing independent evidence that the version row is stale, export the database and record the justification and exact single-row correction. An incorrect version override can cause skipped migrations and irreversible data loss.

**phpMyAdmin trap:** `DATABASE()` may not resolve to the intended site database when exploring `information_schema`; when checking `information_schema.COLUMNS`, confirm the active schema or use an explicit, verified `TABLE_SCHEMA`.

## 4. Shared files require a site-by-site upgrade protocol

Recommended sequence:

```text
Back up everything
  ↓
Stop crons / email workers; put sites into maintenance
  ↓
Deploy and validate the shared, versioned Core files
  ↓
For each site: confirm host → database → schema and plugin state
  ↓
Run Core database migration, then plugin upgrades
  ↓
Verify version, logs, plugin state, paths and public/admin rendering
  ↓
Repeat for the next site
  ↓
Reopen sites one by one; restore reviewed crons; remove public installer
```

Protect shared `siteconfig.php` / `db-config.php` from installer overwrites. Site-specific bootstrap paths must be chosen **before** Core logger, template, cache and other services initialize. See [multisite development principles](multisite-development-principles.md) and [shared-files upgrade safety](plugin-shared-files-upgrade-safety.md).

The upgrade installer should verify the **active site's** directories, not warn about hard-coded default paths from another site's configuration.

## 5. Real failure modes and how to prevent them

### 5.1. Optional Polls table absent: SQL 1146

A Core migration attempted to delete orphan Polls comments even on sites where the Polls plugin had been uninstalled:

```sql
DELETE c FROM gl_comments c
WHERE type = 'polls'
  AND sid NOT IN (SELECT sid FROM gl_polltopics pt WHERE c.sid = pt.pid);
```

The database correctly reported that `gl_polltopics` did not exist. **An optional plugin's tables must not be assumed to exist on every site.**

In Geeklog 2.2.2's MySQL driver, `DB_checkTableExists()` expects the **logical table key**, not the expanded physical table name:

```php
if (DB_checkTableExists('polltopics')) {
    // Queue the orphan-comment cleanup for the current site's Polls table.
}
```

Avoid `DB_checkTableExists($_TABLES['polltopics'])`: internally the driver already looks up `$_TABLES[$tableName]`. Passing the physical name is incorrect.

Do not create a fake Polls table or discard comments merely to satisfy the upgrade script.

### 5.2. Partial failure and unsafe replay

A migration can apply schema changes and then fail **before** recording the final `database_version`. A blind retry may attempt to add the same route or drop the same table twice.

Observed examples included a pre-existing article route and already dropped legacy tables. Safer migration code checks preconditions before inserts, uses `DROP TABLE IF EXISTS` only when semantically appropriate, and performs repeat-safe transformations.

**Important:** not every migration is idempotent. In particular, user-table consolidation can be destructive and should not be replayed casually. On failure:

1. Stop; do not repeatedly reload the installer.
2. Capture the exact new SQL/PHP error with timestamp, hostname and database.
3. Export the current partial state as well as retaining the pre-upgrade backup.
4. Inspect what statements actually completed, including schema objects and rows.
5. Prefer a tested corrective migration or a clean restore and rerun. Adjust version checkpoints manually **only** after verifying every intermediate transformation and obtaining an exact recovery plan.

### 5.3. SpamX plugin upgrade failed: SQL 1091

On some sites, SpamX 1.3.2's upgrade attempted:

```sql
ALTER TABLE gl_spamx DROP PRIMARY KEY;
```

But the table had no primary key; it contained the non-unique `spamx_name` index. This caused `Can't DROP INDEX PRIMARY` during the **plugin upgrade stage**, after Core already reached 2.2.2.

The successful operational workaround in this deployment was to **disable SpamX before running the site upgrade**, preserving its tables and data, then investigate a separate schema-aware SpamX upgrade. This is **not a universal instruction to disable security plugins**: doing so reduces anti-spam protection and requires compensating controls. Audit each site's SpamX structure and upgrade state; restore it only after a validated fix. Never claim an upgrade succeeded merely because its plugin was skipped.

After an installer fails at plugin stage, verify `gl_vars.database_version` first. If it already reads `2.2.2`, do not re-run the complete Core migration.

### 5.4. PHP 5.6 compatibility in likes code

An observed PHP 5.6 failure arose when `DateTime::createFromFormat('U.u', ...)` returned `false` and the code immediately called `setTimeZone()` on the result.

Guard potentially false-returning date parsing and use an explicitly compatible date-construction path. Test the whole upgrade flow on the actual deployed PHP runtime, not only PHP syntax lint.

### 5.5. Custom SQL references removed user tables

Legacy code in `lib-custom.php` referenced `$_TABLES['userinfo']` and `$_TABLES['userprefs']`. Geeklog 2.2.2 consolidated fields such as `showonline` and `lastlogin` in `$_TABLES['user_attributes']`, **not** `$_TABLES['users']`.

A correct modern query must join `users`, `user_attributes` and, if needed, `sessions`. Check real columns and site-specific prefixes before deploying. Use `IS NOT NULL` rather than `!= NULL` in SQL.

Custom PHP blocks, themes and integrations deserve an explicit *post-Core-migration* compatibility audit.

### 5.6. Shared theme and public assets

After reopening the first site, it rendered with an unexpected theme and appeared unstyled. The actual generated HTML revealed the selected layout. A consistent, early theme setting restored the intended shared Eclipse theme.

Check rendered HTML and HTTP status/content type for CSS/JS endpoints; don't assume a theme is active merely because its files are present.

### 5.7. TLS CA bundles for RSS and HTTP clients

RSS feed readers reported `failed loading cafile stream` for missing or invalid site-specific `cacert.pem` files. This is a **local outgoing TLS trust-store configuration problem**, not proof that the website's HTTPS certificate is invalid.

Check the configured `openssl.cafile` / `curl.cainfo` path and available trusted CA bundles. **Do not bypass certificate validation.**

### 5.8. Cron, locking and queued mail

Back up and empty the crontab during maintenance. Review **each** saved job before restoring: an HTTP endpoint and a CLI worker may both process the same email queue.

Use non-overlapping schedules, locks such as `flock`, realistic `timeout` limits, explicit site context, and logs sufficient to confirm successful completion. An empty crontab, an existing lock file or the absence of a running process **does not prove** a queued job succeeded. Confirm digest/email queues are authorized before restarting send workers.

### 5.9. Retire the public installer

After completing the fleet upgrade and testing recovery paths, **remove or move the shared `admin/install` directory out of the public document root**. Re-test the public installer URL and retain a protected archive for future controlled upgrades.

## 6. Per-site acceptance criteria

For every site, record:

| Check | Acceptance condition |
| --- | --- |
| Site routing | Hostname resolves to the intended site database and paths |
| Database | `gl_vars.database_version` matches target release, independently validated |
| Plugins | Previously enabled plugins upgraded or explicitly documented as disabled |
| Storage | Images, uploads, persistent plugin data and logs use correct site directories |
| Frontend | Home, representative article, navigation, assets and feed work |
| Administration | Login, configuration pages and a representative admin action work |
| Errors | No new critical SQL/PHP errors after tests |
| Background jobs | Jobs are restored deliberately and monitored |
| Security | Installer not publicly reachable; compensating controls tracked for skipped plugins |

A public HTTP 503 can be **intentional maintenance**, not an installer failure. A HTTP 200 does not demonstrate a complete upgrade or correct page styling.

## 7. Next-version design and test recommendations

The next Geeklog Core upgrade should ship **automated upgrade preflight and an explicit recovery checkpoint**, especially for multisite:

- Report active site host, current database/schema, declared version, detected schema inconsistencies and plugin migration blockers *before writing*.
- Verify table/column/index preconditions and optional plugin presence; distinguish logical `$_TABLES` keys from physical names.
- Make each individual migration stage resumable, idempotent where possible, and verifiable; record completed stages **per site**.
- Separate Core schema migrations from plugin upgrades; isolate plugin failures with actionable diagnostics rather than an opaque HTTP 500.
- Preserve shared configuration, persistent state and site-specific logs; never overwrite bootstrap configuration implicitly.
- Test absent optional plugin tables, previously removed plugins, unusual historical version rows, partially applied upgrades and PHP 5.6.40.
- Include an automated compatibility audit for historical custom SQL and a final acceptance report for all sites.
- Publish recovery procedures for database restoration and checkpoints; do not normalize manual version-string jumps.

These are **recommendations learned from this deployment**, not a claim that current upstream Geeklog implements these capabilities.

## Related references

- [Multisite development principles](multisite-development-principles.md)
- [Shared-files plugin upgrade safety](plugin-shared-files-upgrade-safety.md)
- [Plugin configuration migration to Geeklog 2.2.2](plugin-configuration-migration-guide-2.2.2.md)
- [Persistent plugin storage](plugin-persistent-storage-guide.md)
- [Current runtime support policy](geeklog-runtime-support-policy.md)

## 8. Proposed maintenance architecture

The historical lessons inform a separate proposal for future upgrade and legacy-SQL recovery. See [Geeklog Install / Upgrade / Rescue architecture](geeklog-installer-upgrade-rescue-architecture.md). The guide above records observed incidents; proposed features must not be mistaken for implemented functionality.
