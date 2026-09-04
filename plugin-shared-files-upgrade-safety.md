# Plugin Shared-Files Upgrade Safety

## Purpose

Geeklog plugins may be deployed in environments where several sites share the same plugin files while each site keeps its own database state, configuration and persistent data.

In that model, uploading a new plugin version updates the PHP files for every site immediately, but each site may run its Geeklog plugin upgrade at a different time.

A modernized plugin must therefore remain operational between these two events:

1. new plugin files are deployed;
2. the current site completes its own plugin upgrade.

This is a release requirement for future plugin versions intended to support shared-files multisite installations.

## Core rule

> **Uploading new shared plugin files must not require every site to upgrade its persisted plugin state at the same time.**

New plugin code must remain compatible with the previous supported persisted state until the upgrade routine for the active site has completed successfully.

The plugin must not assume:

```text
new files present = database/configuration/data already migrated
```

These are separate states.

## Typical multisite deployment

A shared-files installation may temporarily look like this:

```text
shared /plugins/example/ files: 2.0.0

site A persisted plugin state: 2.0.0
site B persisted plugin state: 1.9.0
site C persisted plugin state: 1.9.0
site D persisted plugin state: 2.0.0
```

All four sites execute the same 2.0.0 PHP files.

Sites B and C must continue to operate safely until their administrators run the 1.9.0 -> 2.0.0 upgrade.

## Required runtime behavior

When the new code detects that the current site's persistent state has not yet been upgraded, it should use a temporary compatibility path where necessary.

Recommended model:

```text
current plugin files
        ↓
current site's persisted state upgraded?
 ├─ yes → normal current-version path
 └─ no  → supported legacy compatibility path
              ↓
         no destructive migration
         no cross-site changes
         old data remains usable
```

The exact detection mechanism depends on the plugin. It may use the installed plugin version, existence of a new configuration group, schema markers, storage format markers, or carefully chosen feature detection.

Do not rely only on the version embedded in the files.

## Compatibility path

A compatibility path is not a second permanent implementation. It is a temporary bridge that allows the new files to understand the previous supported persisted state.

Examples include:

- reading an existing legacy configuration file until configuration migration occurs;
- reading an old JSON or serialized data format until conversion occurs;
- tolerating a missing newly-added database column until the site's upgrade creates it;
- falling back to an older rendering or API path while the current site's state still requires it;
- preserving an existing template variable until the site has explicitly migrated to a new rendering mode.

Keep this code isolated and documented so it can be removed after the supported migration window closes.

## Upgrade routines own migrations

Persistent migrations should normally happen inside the plugin's explicit upgrade path, not merely because new PHP files were uploaded.

`plugin_upgrade_PLUGIN()` or the corresponding controlled upgrade routine should be responsible for:

- adding or altering tables and columns;
- creating new configuration values;
- migrating legacy configuration;
- converting persistent files;
- changing storage formats;
- recording the new installed version.

Normal frontend requests should not silently perform destructive schema or data migrations.

A read-only compatibility fallback before upgrade is preferable to an implicit migration triggered by a visitor request.

## Preserve legacy data until success

An upgrade must not remove the previous source of truth before the replacement has been written and validated.

Recommended sequence:

```text
read legacy state
      ↓
validate it
      ↓
write new state
      ↓
verify write succeeded
      ↓
record upgraded state
      ↓
archive legacy data when appropriate
```

If conversion fails, the old data must remain available so the site can recover or retry.

When practical, archive rather than delete the legacy file during the first migration.

## Idempotent and restartable upgrades

Upgrade routines should be safe to retry.

Before creating a table, column, configuration value, directory or converted file, check whether the expected target already exists when the Geeklog API and supported database versions make this practical.

A partially-completed migration should not force manual database surgery before it can be retried.

The upgrade should either:

- complete and record the new state; or
- fail clearly while preserving enough legacy state to retry safely.

## Configuration changes

Adding a new configuration system is a common source of shared-files breakage.

New runtime code should not simply load defaults when the absence of the new configuration actually means "this site has not upgraded yet".

For a migration such as:

```text
legacy PHP configuration
        ↓
Geeklog Configuration API + private JSON
```

the new files should continue to understand the legacy configuration until the current site's upgrade converts it.

After successful conversion, the runtime should use only the new storage for that site.

## Database schema changes

New code must not issue queries that unconditionally require a column or table that exists only after the new upgrade.

Possible strategies include:

- defer use of the new feature until the site has upgraded;
- isolate queries by persisted plugin version or schema capability;
- retain the previous query path during the transition;
- fail the optional new feature gracefully rather than breaking the entire plugin.

Do not attempt to modify another site's schema from the current site.

## Shared code, site-scoped state

The files may be shared, but persistent state belongs to the active site.

Compatibility and migration logic must therefore use the current Geeklog context, including appropriate values from:

```php
$_CONF
$_TABLES
```

and site-scoped configuration and storage paths.

Never scan sibling site databases or directories in order to upgrade them implicitly.

A future Multisite Manager may coordinate upgrades across sites, but individual plugins must remain safe without it.

See [`multisite-development-principles.md`](multisite-development-principles.md).

## Logging

Upgrade and compatibility failures should be diagnosable.

Useful upgrade log information includes:

- plugin name;
- source and target plugin versions;
- active site context when safely identifiable;
- migration stage that failed;
- legacy file or storage path involved when appropriate;
- whether legacy data was preserved.

Avoid noisy logs on every normal request simply because a site has not upgraded yet.

## Release checklist

Before releasing a plugin version that changes persistent state, test all of the following:

- fresh installation of the new version;
- normal upgrade from the previous supported version;
- new files uploaded while the site still has the previous persisted state;
- frontend requests before the upgrade is run;
- administration requests before the upgrade is run;
- successful upgrade after this transitional period;
- failed or interrupted migration where practical;
- retry of the upgrade;
- at least two multisite contexts sharing plugin files but using different persisted plugin versions;
- confirmation that upgrading site A does not alter site B's configuration, files or database state.

## Anti-patterns

Avoid these patterns:

```text
New files immediately query a new column that old sites do not have.
```

```text
New runtime code ignores legacy configuration and silently falls back to defaults.
```

```text
Loading the plugin automatically rewrites persistent data before an explicit upgrade.
```

```text
The first upgraded site deletes or converts storage shared by sites that have not upgraded.
```

```text
The upgrade marks the new version installed before all required migrations succeed.
```

## Example: configuration migration

Suppose plugin 1.0 stores rules in:

```text
{path_data}/plugin_config.php
```

and plugin 1.1 introduces Geeklog Configuration plus private JSON.

The safe lifecycle is:

```text
1.1 files uploaded
        ↓
site still recorded as 1.0
        ↓
1.1 runtime reads legacy configuration through compatibility layer
        ↓
site continues to work
        ↓
administrator runs upgrade
        ↓
legacy configuration converted
        ↓
new state verified
        ↓
installed version becomes 1.1
        ↓
legacy file archived
        ↓
compatibility layer no longer used for this site
```

This allows other sites sharing the same 1.1 files to remain on their 1.0 persisted state until they are upgraded individually.

## Developer policy

For plugins maintained under the current modernization effort, shared-files upgrade safety should be considered during design whenever a release changes:

- configuration storage;
- database schema;
- persistent files;
- template integration;
- identifiers or URL formats;
- permissions or groups;
- lifecycle state required by runtime code.

The preferred order is:

> **Deploy files safely → preserve legacy operation → upgrade one site → verify migration → switch that site to the new state.**

## Guiding principle

> **A plugin upgrade is a site-scoped state transition, not a side effect of replacing shared files.**
