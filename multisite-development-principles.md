# Geeklog Multisite Development Principles

## Purpose

Multisite support is a development constraint that applies now, even though a dedicated Multisite Manager plugin is a later project.

Current plugin modernization should remain compatible with:

- **Geeklog 2.1.1 through 2.2.2**
- **PHP 5.6 through PHP 8.1**

## Core principle

A plugin must operate within the context of the current Geeklog site and must not assume that one installation equals one site.

Every plugin storing configuration, files or persistent state should define:

- site context;
- data isolation;
- configuration isolation;
- permissions context;
- installation behavior;
- upgrade and migration behavior.

## Site context

Use the configuration already selected by Geeklog for the active site.

Plugins should prefer site-scoped values such as:

```php
$_CONF['path_data']
$_CONF['path_html']
$_CONF['site_url']
$_CONF['site_admin_url']
```

rather than recreating host detection or maintaining a second site-selection mechanism inside each plugin.

## Persistent data isolation

Persistent files should derive their storage path from the current site's configuration.

Do not use one shared hard-coded directory for all sites unless shared data is an explicit feature.

See [`plugin-persistent-storage-guide.md`](plugin-persistent-storage-guide.md).

## Database isolation

Plugins must use the active Geeklog table definitions from `$_TABLES` and must not hard-code prefixes such as `gl_`.

Where a multisite installation uses separate tables or different prefixes, the plugin should follow the table mapping already established for the active site.

A plugin should not attempt to discover or modify another site's tables during normal frontend or administration requests.

Cross-site administration belongs in a dedicated master-site tool such as the planned Multisite Manager.

## Configuration isolation

Use Geeklog's configuration system where appropriate and ensure settings are read from the active site context.

Do not cache plugin configuration globally in a way that can leak values between sites in the same runtime or shared storage environment.

## Permissions

Permissions must always be evaluated in the active site's Geeklog security context.

A user being an administrator on one site must not implicitly grant access to another site's plugin data unless the multisite architecture explicitly defines that relationship.

## Installation and upgrades

Installation and upgrade routines should operate only on the current site's plugin state unless they are explicitly part of a master-site administration operation.

When migrating files or configuration:

- derive paths from the active site;
- avoid scanning sibling site directories;
- do not overwrite another site's data;
- make migrations repeatable where practical;
- log enough context to identify the affected site.

### Shared plugin files and staggered upgrades

A multisite installation may share one plugin directory while each site stores its own installed plugin version, configuration and persistent data.

In that situation, replacing the shared plugin files updates the executable code for every site immediately, but each site may complete its Geeklog plugin upgrade later and independently.

New plugin code must therefore remain operational with the previous supported persisted plugin state until the active site's upgrade has completed successfully.

Uploading new shared files must not require every site to run its migration at the same time.

The full policy, implementation patterns and release checklist are defined in [`plugin-shared-files-upgrade-safety.md`](plugin-shared-files-upgrade-safety.md).

## Compatibility layer

Current plugins may need compatibility code because Geeklog 2.1.1 and 2.2.2 do not expose exactly the same facilities.

Keep compatibility logic isolated where practical.

A recommended pattern is:

```text
feature detection
      ↓
newer Geeklog path when available
      ↓
legacy-compatible fallback when required
```

Avoid version checks when reliable feature detection is possible.

## Multisite Manager status

The future Multisite Manager is a separate concern.

Its purpose may include:

- registering secondary sites;
- enabling or disabling sites;
- managing site-specific configuration;
- provisioning database tables;
- reducing manual edits to `db-config.php` and `siteconfig.php`;
- reusing official Geeklog installation functions where possible.

It is not required before plugins can follow correct multisite principles.

## Testing baseline

A plugin claiming multisite-safe behavior should be tested with at least two site contexts that have different:

- `path_data` values;
- site URLs;
- plugin storage directories;
- database table mappings or prefixes where applicable.

Actions performed on site A must not alter site B's files, configuration or records.

For releases that change persistent plugin state, also test a staggered-upgrade scenario where both sites execute the new shared plugin files but only one site has completed the new plugin migration. The site still using the previous persisted state must continue to operate safely.

## Guiding principle

> Multisite safety should emerge from normal site-scoped plugin design, not from special-case fixes added after a plugin is complete.
