# Geeklog Plugin Development Guide

A practical, end-to-end guide for building a Geeklog plugin that is installable, activatable, deactivatable, configurable, administrable, publicly usable, upgradeable, packageable, and compliant with the current Memorandum development expectations.

This document is intentionally **transversal**. The repository already contains detailed references for individual topics such as the Plugin API, configuration migration, administration navigation, persistent storage, multisite behavior, shared-file upgrades, and interoperability. This guide explains **in which order those pieces should be assembled** to produce an operational plugin.

## Design principle: simple and robust by default

Prefer the **simplest robust design that solves the shared problem**.

Do not preserve a fragile architecture by accumulating route-specific, version-specific, provider-specific, page-specific, or theme-specific patches around it.

When repeated exceptions appear, stop and examine the underlying contract, data model, lifecycle, or ownership boundary.

A modernized plugin should favor:

- one source of truth;
- one canonical definition for each Plugin API callback;
- one generic provider contract instead of consumer-side branches for individual plugins;
- one clear lifecycle path for install, upgrade, enable, disable, and uninstall;
- shared helpers only when they remove real duplication or coupling;
- isolated compatibility code when an older Geeklog version genuinely requires it;
- removal of obsolete paths once their replacement is proven;
- explicit ownership of data, rendering, URLs, permissions, and business rules;
- small coherent refactors over chains of local corrective patches.

Avoid both extremes:

- **patch accumulation** — many local fixes that hide the same structural problem;
- **premature abstraction** — extra layers introduced before they solve a demonstrated duplication or interoperability need.

A useful warning sign is:

> **Three exceptions to the same rule usually mean the shared design should be revisited.**

Compatibility code is sometimes necessary, especially across Geeklog 2.1.1–2.2.2, but it should be narrow, documented, testable, and removable when the compatibility boundary disappears.


## Current compatibility baseline

Unless a plugin repository explicitly defines another policy, the current modernization target is:

- Geeklog 2.1.1 through 2.2.2;
- PHP 5.6 through PHP 8.1.

New code must therefore use the common safe PHP subset required by that range and must not assume that every Geeklog 2.2.2 facility exists unchanged in Geeklog 2.1.1.

See:

- [Plugin API reference](plugin-api-reference-2.2.2.md)
- [Configuration migration guide](plugin-configuration-migration-guide-2.2.2.md)
- [Shared-files upgrade safety](plugin-shared-files-upgrade-safety.md)
- [Plugin asset loading and cache versioning](plugin-asset-loading-versioning.md)
- [Multisite development principles](multisite-development-principles.md)

---

# 1. Start with the plugin contract

Before creating files, define the smallest useful plugin.

Write down:

- short plugin name, lowercase where practical;
- human-readable name;
- initial version;
- supported Geeklog versions;
- supported PHP versions;
- whether the plugin owns content;
- whether it needs one or more database tables;
- whether it needs persistent files;
- required permissions/features;
- required administration pages;
- required public pages;
- required configuration options;
- whether it exposes blocks, search, feeds, sitemap content, Item Info, lifecycle events, services, or other integrations.

Do not begin by implementing every possible Geeklog callback. Implement the smallest coherent surface first, then add optional capabilities because the plugin genuinely needs them.

A useful initial question is:

> What must exist for an administrator to install the plugin, configure it, create or manage something, and for an authorized user to use the result?

---

# 2. Create the minimum plugin structure

A typical plugin should have a structure comparable to:

```text
PLUGIN/
├── autoinstall.php
├── functions.inc
├── version.php
├── install_defaults.php
├── language/
│   ├── english.php
│   └── ...
├── admin/
│   └── index.php
├── public_html/
│   └── index.php
├── templates/
│   └── ...
├── sql/
│   └── ...
├── css/
│   └── ...
└── js/
    └── ...
```

Not every directory is mandatory. Add only what the plugin uses.

Common responsibilities:

- `version.php` — plugin version and compatibility metadata used by the project;
- `functions.inc` — Plugin API callbacks and bootstrap-safe shared code;
- `autoinstall.php` — installation/uninstallation metadata and lifecycle;
- `install_defaults.php` — configuration defaults where the plugin uses Geeklog Configuration;
- `admin/` — administrative entry points;
- `public_html/` — public entry points;
- `templates/` — presentation markup;
- `sql/` — install or migration SQL when appropriate;
- `css/`, `js/` — plugin-owned assets.

Plugin-owned static assets must not stop at “the file exists”. They should be external files, loaded only on relevant pages/contexts where practical, and use deterministic cache versioning. For administration assets, prefer a clear path such as `admin/css/admin.css` or `css/admin.css`, load it through the normal Geeklog header integration, and verify the asset is present in the packaged ZIP.

See [Plugin asset loading and cache versioning](plugin-asset-loading-versioning.md).

Keep unconditional bootstrap code in `functions.inc` small. Any fatal error there can break every page on which Geeklog loads the plugin.

## One canonical definition per Plugin API callback

Before adding or moving a callback, search the **entire plugin tree** for an existing implementation.

This is especially important in older plugins where callbacks may be split between files such as:

```text
functions.inc
api.inc
modern.inc
lib-*.php
```

A callback such as:

```php
plugin_getadminoption_PLUGIN()
plugin_cclabel_PLUGIN()
plugin_getconfigtooltip_PLUGIN()
plugin_getiteminfo_PLUGIN()
plugin_idtourl_PLUGIN()
```

must have one canonical runtime definition.

Do not "add the modern callback" to a new file until confirming that a historical implementation does not already exist elsewhere. Duplicate definitions commonly produce a site-wide fatal error during plugin bootstrap.

Recommended practice:

1. search the complete source tree for the callback name;
2. choose the canonical implementation location;
3. remove or consolidate older duplicates;
4. ensure the canonical file is loaded in every runtime path that needs the callback;
5. add a CI/source-contract check for important callbacks when the plugin has a history of duplicate definitions.

---

# 3. Define database ownership before installation code

If the plugin owns database tables, define them explicitly and consistently.

## Use `$_TABLES`

Register plugin tables through Geeklog's table map and use those entries everywhere.

Example concept:

```php
$_TABLES['myplugin_items'] = $_DB_table_prefix . 'myplugin_items';
```

Then query:

```php
DB_query("SELECT * FROM {$_TABLES['myplugin_items']}");
```

Do not hardcode an installation prefix such as `gl_`.

Bad:

```php
SELECT * FROM gl_myplugin_items
```

Good:

```php
SELECT * FROM {$_TABLES['myplugin_items']}
```

## Keep schema ownership inside the plugin

Other plugins should not need to know your private table names or column names.

If content must be consumed elsewhere, expose it through Geeklog Plugin APIs or documented provider contracts.

A consumer should not read another third-party plugin's private tables merely because a shared capability is temporarily incomplete. If a compatibility audit must read Geeklog-owned Core tables because Core/Static Pages do not expose an equivalent contract on a supported version, keep that fallback:

- explicitly limited to Geeklog-owned tables;
- read-only where practical;
- permission- and language-aware;
- isolated and documented for later removal;
- never generalized into arbitrary third-party SQL access.

See [Plugin content interoperability contract](plugin-content-interoperability-contract.md).

## Plan schema evolution now

Even for version 1.0.0, decide how future schema changes will be applied.

A plugin with tables but no upgrade strategy is incomplete.

---

# 4. Make installation and rollback work before building features

Installation is not complete merely because a clean install succeeds once.

The plugin should define:

- tables;
- groups;
- features/permissions;
- configuration defaults;
- plugin variables where required;
- PHP blocks if the plugin creates them;
- any other plugin-owned install-time state.

It should also implement a complete `plugin_autouninstall_PLUGIN()` contract so Geeklog can clean up both:

- a normal uninstall;
- a partially failed installation.

## Required installation tests

Test at least:

1. clean installation;
2. activation;
3. deactivation;
4. reactivation;
5. uninstall;
6. reinstall;
7. intentional failure after some install objects have been created;
8. cleanup of that partial state;
9. second installation after the failed attempt.

A failed installation must not leave tables, features, groups, configuration records, or plugin variables that cause duplicate-entry errors on the next attempt.

## Installation should be repeatable

Where practical, initialization and migration helpers should be idempotent:

- create what is missing;
- detect what already exists;
- do not duplicate records;
- do not destroy user data to repair a partially initialized state.

## Autoinstall scope: declare plugin globals explicitly

Geeklog 2.1.1 may load plugin files such as `functions.inc` and `install_defaults.php` from inside the core autoinstall function `plugin_do_autoinstall()`.

In PHP, a file included from inside a function inherits that function scope. Plugin variables that normally appear to be global when loaded during standard bootstrap are therefore **not automatically available as globals during autoinstall**.

This can produce failures such as:

```text
array_merge(): Argument #1 is not an array
```

when code assumes that a global default/configuration array already exists.

For bootstrap/install files that depend on plugin-owned globals, declare them explicitly before use:

```php
global $_CONF;
global $_MYPLUGIN_DEFAULT, $_MYPLUGIN_CONF;
global $LANG_MYPLUGIN, $LANG_MYPLUGIN_ADMIN;
```

The same rule applies to any plugin-owned arrays populated by language or configuration files.

Do not rely on `require_once` to solve scope. `require_once` prevents duplicate inclusion, but it does not promote variables into global scope. During autoinstall, it can actually hide the problem if a file was already included in another scope.

Safer pattern:

```php
global $_CONF, $_MYPLUGIN_DEFAULT, $_MYPLUGIN_CONF;
global $LANG_MYPLUGIN;

$pluginPath = $_CONF['path'] . 'plugins/myplugin/';

if (!isset($LANG_MYPLUGIN) || !is_array($LANG_MYPLUGIN)) {
    require $pluginPath . 'language/english.php';
}

require_once $pluginPath . 'install_defaults.php';

if (!isset($_MYPLUGIN_DEFAULT) || !is_array($_MYPLUGIN_DEFAULT)) {
    $_MYPLUGIN_DEFAULT = array();
}
```

Important distinctions:

- use explicit `global` declarations for plugin-owned state needed during bootstrap/install;
- guard expected arrays with `isset()` + `is_array()`;
- do not assume a language/config file included earlier populated the current scope;
- keep `functions.inc` bootstrap-safe because Geeklog may load it from several lifecycle paths;
- test the real upload/autoinstall path on Geeklog 2.1.1, not only normal page bootstrap.

A plugin that works after manual file placement can still fail during archive upload if this scope behavior is not tested.

---

# 5. Make activation and deactivation safe

Installation, activation, deactivation, and uninstallation are different lifecycle states and must not be conflated.

A plugin should support:

```text
installed + enabled
installed + disabled
uninstalled
```

Disabling a plugin must **not** destroy its data, configuration, tables, files, or upgrade state.

When disabled:

- public functionality must no longer be exposed through normal Geeklog plugin dispatch;
- administration/menu integration should disappear according to Geeklog behavior;
- scheduled or indirect plugin work must not continue merely because files still exist;
- direct access to plugin PHP entry points must fail safely when the plugin is inactive;
- plugin-owned persistent data must remain intact for later reactivation.

When re-enabled:

- the plugin should resume from its previous persisted state;
- it must not recreate duplicate groups, features, configuration entries, blocks, or tables;
- it must not silently reset administrator configuration;
- it must not require reinstalling merely because it was disabled;
- it should detect when the installed persisted version requires an upgrade before normal operation continues.

Activation/deactivation tests should include:

1. install and configure;
2. create representative data;
3. disable the plugin;
4. verify public/admin integrations are unavailable as expected;
5. verify persisted data remains intact;
6. re-enable the plugin;
7. verify configuration and content are preserved;
8. verify normal operation resumes without duplicate initialization.

A plugin that can only be installed and uninstalled, but cannot survive disable/re-enable cleanly, is not lifecycle-complete.

---

# 6. Add configuration using Geeklog Configuration

If a value belongs to site administration, use Geeklog's Configuration API rather than inventing an unrelated settings store.

A useful configuration hierarchy is:

```text
plugin
└── subgroup
    └── tab
        └── fieldset
            └── option
```

Configuration implementation commonly involves:

- `install_defaults.php`;
- language labels/help;
- selection arrays where appropriate;
- default values;
- migration logic for existing installations.

## New install and upgrade are different problems

Adding a new default to `install_defaults.php` only solves **new installations**.

Existing sites also need that option inserted during an upgrade.

Therefore:

> Adding a configuration option to an already released plugin is an upgrade change.

Normally this means:

- bump the plugin version;
- add an idempotent configuration migration;
- verify old installations receive the new key;
- verify already-upgraded sites do not receive duplicates.

A helper such as `ensureConfig()` can be useful when its behavior is deterministic and idempotent.

For technical or consequential settings, consider Geeklog's native contextual help callback:

```php
plugin_getconfigtooltip_PLUGIN($id)
```

Keep tooltip text in language files, return an empty string safely for unknown keys, and verify that this callback is defined only once in the plugin.

See:

- [Configuration migration guide](plugin-configuration-migration-guide-2.2.2.md)
- [Plugin configuration tooltips](plugin-configuration-tooltips.md)

---

# 7. Build administration as a protected application surface

The administration page should not rely on the invisibility of a menu link for security.

At every administrative entry point:

- verify the required Geeklog feature with `SEC_hasRights()` or the appropriate ACL mechanism;
- reject unauthorized access before doing useful work;
- protect state-changing forms/actions with Geeklog CSRF tokens;
- validate request input;
- authorize mutations again on the server;
- escape rendered output for its context.

## Administration navigation

Provide a clear plugin-local administration structure when the plugin has several administration sections.

Typical sections may include:

- main management page;
- categories;
- import/export;
- diagnostics;
- configuration;
- maintenance.

Use the repository's shared navigation conventions rather than inventing a different UI for every plugin.

A normal administration area should also be discoverable through Geeklog's native plugin administration surfaces. Where appropriate, implement both:

```php
plugin_getadminoption_PLUGIN()
plugin_cclabel_PLUGIN()
```

The first feeds Geeklog's administration menu; the second feeds Command & Control. Themes and dashboards should consume these native entries rather than hard-code plugin URLs.

See:

- [Plugin admin navigation](plugin-admin-navigation.md)
- [Plugin administration UX guidelines](plugin-admin-ux-guidelines.md)

## First-use documentation

For a plugin with more than trivial administration, provide a short installed Getting started / Help path.

A new administrator should be able to understand:

1. what the plugin does;
2. what must be configured first;
3. how to create or manage the primary object;
4. where the result appears publicly;
5. what to check when the expected result does not appear.

This can be a short block on the administration home page, a collapsible help section, or a dedicated Help page. Keep it concise and aligned with the shipped version.

## Configuration access

If the plugin exposes configuration, administration should make it easy to reach.

Do not force administrators to discover the plugin's configuration through unrelated Geeklog pages.

## Avoid large HTML blobs in PHP

For substantial administration interfaces:

- keep request/controller logic in PHP;
- keep repeated or significant markup in `.thtml` templates;
- keep JavaScript and CSS in dedicated files.

This improves maintainability and theme compatibility.

---

# 8. Make the public entry point work early

Do not postpone public integration until the end.

A content-facing plugin should normally have a public entry point such as:

```text
/public_html/index.php
```

The page should:

- load Geeklog through the normal bootstrap;
- enforce relevant ACL;
- build the page content;
- render a complete document through `COM_createHTMLDocument()`;
- behave correctly when the plugin has no content;
- behave correctly for anonymous users;
- behave correctly for logged-in users;
- avoid direct dependence on a specific theme.

For the current Geeklog 2.1.1–2.2.2 compatibility range, modernized full pages should use `COM_createHTMLDocument()`, not legacy `COM_siteHeader()` / `COM_siteFooter()` rendering.

## Public design, usability and SEO

A public page is not complete merely because it renders without errors.

For every significant public surface, review:

- information hierarchy and readability;
- responsive/mobile behavior;
- accessibility and keyboard use;
- empty/error states;
- canonical identity and URL;
- document title and meta description;
- indexability;
- crawlable internal links;
- structured data and social metadata where appropriate.

See:

- [Plugin public design guidelines](plugin-public-design-guidelines.md)
- [Plugin SEO public page guidelines](plugin-seo-public-page-guidelines.md)

## Public navigation

If users need to reach the plugin from the site navigation, implement the appropriate Plugin API menu callback, for example:

```php
plugin_getmenuitems_PLUGIN()
```

A perfectly functional `/plugin/index.php` that nobody can discover is not a complete public integration.

## Disabled plugin behavior

Public entry points should fail safely when the plugin is disabled or inaccessible.

Do not expose mutations or private data merely because a PHP file remains addressable directly.

---

# 9. Add blocks only when they have a real user role

If the plugin provides reusable dynamic blocks, expose them through Geeklog's block integration rather than hardcoding theme-specific widgets.

For plugins where this is useful, consider:

```php
plugin_getBlocks_PLUGIN()
```

Distinguish between:

- normal Geeklog blocks;
- centerblock-style presentation;
- plugin-specific page components.

Do not implement a dynamic block merely to satisfy a checklist.

A block should have a meaningful purpose such as:

- recent items;
- featured item;
- status;
- compact player;
- category navigation;
- plugin summary.

---

# 10. Load CSS and JavaScript through the plugin lifecycle

Keep CSS and JavaScript in dedicated files and register them through compatible Geeklog mechanisms.

When plugin callbacks are used for asset injection, common patterns include:

```php
plugin_getheadercode_PLUGIN()
plugin_getfootercode_PLUGIN()
```

Do not assume arbitrary keys such as `footercode` passed to `COM_createHTMLDocument()` are equivalent to the Plugin API callback.

## Match both directory and index routes

A common bug is loading assets only when the URL literally ends with:

```text
/plugin/index.php
```

while the site actually exposes:

```text
/plugin/
```

Asset/page detection must correctly handle both forms when both are valid.

## Version CSS and JavaScript assets

Browsers cache CSS and JavaScript aggressively. Plugin CSS and JavaScript files should therefore be **versioned in their public URL**.

A recommended convention is to derive the asset version from:

- the plugin version;
- and, where useful, `filemtime()` for the actual file.

Example concept:

```text
plugin.css?v=1.4.0-1727600000
plugin.js?v=1.4.0-1727600000
```

Requirements:

- CSS and JS changes must invalidate the browser cache after deployment;
- the version must change deterministically when the shipped asset changes;
- do not rely on users clearing browser cache manually;
- do not use random query strings on every request;
- do not leave static, unversioned asset URLs when the file is expected to change across plugin releases.

This applies to both public and administration assets.

---

# 11. Implement CRUD with validation, ACL, and lifecycle behavior

For create/update/delete operations:

- normalize identifiers;
- validate input;
- enforce ACL before mutation;
- use Geeklog database abstraction;
- escape SQL string values appropriately;
- use transactions where a multi-step mutation must remain atomic;
- protect state changes with security tokens;
- redirect or render a deterministic result;
- emit lifecycle events only after the mutation actually succeeds.

For content plugins, successful saves and deletes should normally consider:

```php
PLG_itemSaved($itemId, 'plugin');
PLG_itemDeleted($itemId, 'plugin');
```

These events allow consumers to react without reading private plugin tables.

---

# 12. Add Geeklog integrations deliberately

Once the plugin's core lifecycle works, decide which native integrations make sense.

Possible capabilities include:

- Search;
- What's New;
- public menu;
- blocks;
- statistics;
- Content Syndication;
- XML Sitemap;
- autotags;
- related items;
- Item Info;
- canonical URL resolution;
- lifecycle events;
- `PLG_itemDisplay()` public extension point;
- services.

Do not implement all callbacks mechanically.

## Content plugins: recommended interoperability baseline

Modernized content plugins should normally consider:

```php
plugin_getiteminfo_PLUGIN()
PLG_itemSaved()
PLG_itemDeleted()
```

and collection support through Item Info when appropriate.

Where supported and useful, also consider:

```php
plugin_idtourl_PLUGIN()
plugin_collectSitemapItems_PLUGIN()
plugin_getfeednames_PLUGIN()
plugin_getfeedcontent_PLUGIN()
plugin_whatsnewsupported_PLUGIN()
plugin_getwhatsnew_PLUGIN()
plugin_dopluginsearch_PLUGIN()
plugin_getrelateditems_PLUGIN()
plugin_getBlocks_PLUGIN()
```

For full public item pages, content providers should also consider the generic extension point:

```php
PLG_itemDisplay($id, $type)
```

## Addressable resources are not limited to leaf items

A modern content provider may expose stable public resources such as:

```text
root / catalogue
category
album
forum
channel
map
marker
topic
terminal item
```

Consumers should not need provider-specific SQL or routing knowledge to address those objects.

When one provider exposes several object families:

- expose a stable `subtype` where supported;
- keep the `id` unambiguous even on APIs that transport only `type + id`;
- prefer provider-owned IDs such as `category:12`, `album:45`, or `forum:8`;
- expose `title` and `url` through Item Info;
- expose canonical URL resolution through `plugin_idtourl_PLUGIN()` where supported;
- do not model POST-only, cookie-only, or session-only UI state as a stable resource; an addressable container should have a canonical provider-owned GET URL;
- consider optional `is-container`, `parent-id`, and `parent-subtype` fields for lightweight hierarchy.

This keeps the same resource usable on Geeklog 2.1.1 while allowing Geeklog 2.2.2 to carry richer subtype information.

## Keep provider families distinct

Not every structured plugin surface is content.

Use the shared capability/provider-family distinction:

```text
content       -> addressable resources / Item Info
navigation    -> permission-filtered tree contracts
relationship  -> Hub/context services
service       -> bounded specialized services
```

Do not force navigation trees, diagnostics or service responses into Item Info merely so one consumer can read them.

## Consumer rule: never hard-code another plugin's routing

A consumer such as FAQ, Hub, Agent, Hello or IndexNow should not contain logic such as:

```php
if ($provider === 'documents') { ... }
if ($provider === 'maps') { ... }
```

to derive another plugin's title or public URL.

Prefer, in order:

1. provider-owned canonical URL resolution such as `plugin_idtourl_PLUGIN()` where available;
2. Item Info fields such as `title` and `url`;
3. a documented compatibility fallback only for Core/legacy surfaces that do not yet expose the shared contract.

Third-party plugin tables and private routing rules must remain private.

See:

- [Plugin API reference](plugin-api-reference-2.2.2.md)
- [Plugin content interoperability contract](plugin-content-interoperability-contract.md)
- [Plugin capability contract](plugin-capability-contract.md)

## Validate against real consumers, not only the written contract

Memorandum compliance is not proven by implementing callbacks whose names and signatures look correct.

When a maintained consumer already exists, test the provider through that consumer's real runtime path.

FAQ is the current reference case for contextual content associations. A provider intended to participate in FAQ associations should be tested for all of the following:

```text
provider discovery
    -> plugin_getiteminfo_PLUGIN() exists and is loaded

association picker
    -> PLG_getItemInfo(PLUGIN, '*', 'id,title,url,subtype,type', ...)

stored association display
    -> PLG_getItemInfo(PLUGIN, id, 'title', current_user_uid)
    -> plugin_idtourl_PLUGIN(subtype, id)
       or PLG_getItemInfo(PLUGIN, id, 'url', current_user_uid)

public contextual rendering
    -> provider calls PLG_itemDisplay(stable_id, PLUGIN)
```

Do not declare the provider aligned merely because a direct unit call to `plugin_getiteminfo_PLUGIN()` returns something plausible.

The acceptance test must prove that:

- the dispatcher reaches the callback;
- the requested field form is supported;
- permissions are evaluated under the intended UID;
- the stable ID is the same across collection, resolution, URL and rendering paths;
- container and leaf ACL/publication state remain consistent across discovery and rendering;
- the consumer does not need a provider-specific exception.

See the **FAQ contextual associations reference consumer profile** in [Plugin content interoperability contract](plugin-content-interoperability-contract.md).

---

# 13. Respect exact API contracts

Do not infer a callback's return format from its name.

Geeklog callbacks may expect:

- positional arrays;
- associative arrays;
- HTML;
- booleans;
- status codes;
- a precise parameter order;
- special collection behavior.

For example, Item Info must follow the exact Geeklog contract used by the supported versions. Returning a convenient custom associative structure where Geeklog expects another representation can appear to work in custom code while breaking native consumers.

When implementing a callback:

1. verify the Plugin API reference;
2. inspect existing working core/plugin implementations when necessary;
3. test the callback through Geeklog, not only by calling your function directly.

## Normalize historical Item Info return shapes at consumer boundaries

Across older and modernized plugins, a concrete Item Info request may be encountered as:

- a scalar for one requested field;
- a positional/numeric array;
- an associative array.

A generic consumer should normalize these forms in one shared helper instead of adding provider-specific exceptions.

When retrieving several fields, preserve the requested field order when mapping a positional result.

## Use the correct permission context

The `$uid` passed to `PLG_getItemInfo()` is part of the permission contract.

Do not automatically use anonymous UID `0` from an authenticated administration page. When an administrator is selecting or inspecting provider content, use the current authenticated user's UID unless the operation intentionally audits public/anonymous visibility.

Conversely, public machine/discovery endpoints should use the visibility context they actually promise and must not leak administrator-only content.

---

# 14. Store persistent data outside disposable cache

Do not place uploads or plugin-owned persistent data in Geeklog cache directories.

Cache is disposable by definition.

Persistent plugin files should be derived from the active site's persistent data location, normally using the site's `path_data` context.

Requirements:

- site-aware path;
- directory creation handled safely;
- web access denied unless explicitly intended;
- downloads served through controlled PHP when authorization is required;
- filenames sanitized/generated safely;
- database records and file lifecycle kept consistent.

See [Plugin persistent storage guide](plugin-persistent-storage-guide.md).

---

# 15. Treat multisite as a constraint from day one

Even when the plugin is not a multisite-management plugin, persistent state should have a clear site context.

Define:

- which database belongs to the active site;
- where configuration is stored;
- where persistent files are stored;
- what happens when plugin code is shared by several sites;
- how permissions are evaluated;
- how one site's upgrade affects another site using the same code files.

## Shared plugin files are especially important

When several Geeklog sites share one plugin directory, deploying new plugin files must not immediately require every site's persisted state to already be upgraded.

New code should remain compatible with the previous supported persisted state until the active site's upgrade is explicitly run.

This requirement also applies to **runtime extension callbacks**. If a plugin is a consumer of `PLG_itemDisplay()`, Item Info, lifecycle hooks, or another shared dispatcher, newly deployed code must not assume that its own database already contains every column introduced by that code. A failure in the consumer can otherwise break an unrelated provider page that merely called the standard Geeklog API correctly.

During a supported transition, keep schema compatibility centralized in the owning plugin's data-access layer:

- detect the actual owned schema when a newer column is optional during the upgrade window;
- resolve one safe SQL expression/default for each missing field;
- use that resolved expression consistently throughout the query;
- degrade gracefully when contextual data cannot be read safely;
- never push consumer-version checks or consumer-specific workarounds into providers.

The upgrade must still normalize the database to the current schema. Runtime compatibility is a safety boundary, not a substitute for migration.

See:

- [Multisite development principles](multisite-development-principles.md)
- [Shared-files upgrade safety](plugin-shared-files-upgrade-safety.md)

---

# 16. Build upgrades as first-class code

Every released state creates a possible upgrade starting point.

An upgrade may need to change:

- table schema;
- indexes;
- stored values;
- configuration entries;
- plugin variables;
- permissions/features;
- persistent file layout;
- derived metadata.

Upgrade routines should be:

- sequential;
- version-aware;
- idempotent where practical;
- non-destructive;
- safe to rerun after interruption;
- compatible with shared-file multisite deployment.

Do not delete legacy data until the replacement has been verified.

## Version changes matter

If the persisted state must change, the plugin needs a recognizable upgrade boundary.

Do not silently add a required database/configuration change while leaving the released plugin version unchanged.

## Upgrade must be a supported lifecycle path

A plugin should not merely contain migration snippets. It should provide a coherent upgrade path from supported released versions.

The upgrade path should:

- determine the currently installed plugin version;
- apply required migrations in order;
- update database schema safely;
- add or transform configuration without resetting user choices;
- add new groups/features/permissions where required;
- migrate persistent files when required;
- update plugin metadata/version state only after the relevant migration succeeds;
- stop safely and report a useful error when a migration cannot complete;
- remain safe when the same code files are shared by several sites at different persisted versions.

Where Geeklog exposes a native plugin upgrade mechanism or callback for the supported release, use that lifecycle instead of inventing an unrelated upgrade endpoint.

After an upgrade, test both:

- the upgraded installation;
- a clean installation of the same new version.

They must converge on an equivalent supported state.

## Upgrade matrix

For each release, record which previous versions are directly supported upgrade sources.

At minimum, test the immediately previous released version. When older supported versions can legitimately upgrade directly, include them in the test matrix.

Never assume:

```text
latest files = latest database/configuration
```

The plugin must explicitly reconcile code version and persisted state.

---

# 17. Diagnose blank pages from the bootstrap outward

A blank page is often not a template problem.

If both administration and public pages fail, inspect the common plugin bootstrap first.

Check:

1. `php -l` on every shipped `.php` and `.inc` file;
2. server `error.log`;
3. unconditional includes from `functions.inc`;
4. unsupported PHP syntax;
5. missing configuration keys used during bootstrap;
6. undefined request/session/configuration indexes;
7. language arrays used before loading;
8. `die()`, `exit`, guards, or early returns in globally loaded files;
9. legacy rendering calls incompatible with the target Geeklog version.

Do not hide the root cause with `@`.

A distribution containing a PHP syntax error should never reach `dist/`.

---

# 18. Package the plugin as an installable artifact

The development tree is not automatically a valid release archive.

The archive should contain exactly the plugin structure Geeklog expects.

Check:

- correct root plugin directory;
- versioned archive name;
- no `.git`, `.github`, editor metadata, local test files, or dotfiles unless intentionally required;
- no development-only fixtures;
- required language/templates/assets included;
- executable PHP files syntax-checked;
- archive can be installed on a clean Geeklog instance.

A useful naming pattern is:

```text
plugin_VERSION_GEEKLOG.zip
```

## Release version names must be stable

When a branch or archive represents a concrete release such as `1.4.0`, the plugin version reported to Geeklog must be exactly that release version:

```text
1.4.0
```

Do not publish runtime version names such as:

```text
1.4.0-dev
1.4.0-final
1.4.0-release
```

unless the project has explicitly adopted a prerelease/versioning scheme that Geeklog and all upgrade logic are designed to understand.

For normal Geeklog plugin releases, keep the canonical version simple and stable across:

- `autoinstall.php` / `pi_version`;
- `plugin_chkVersion_PLUGIN()`;
- upgrade comparisons;
- README/release notes;
- archive naming;
- displayed administration metadata.

Development state belongs in branch names, roadmap/status text, issues, milestones, or release notes — **not in the canonical runtime plugin version**.

## Branch discipline: never assume the default branch is the work branch

A repository's default branch (often `main`) is not automatically the branch on which current development must be written.

Before **any repository write**:

1. identify the explicit active development branch from the user's instruction, repository documentation/roadmap, or the already established development context;
2. verify that the branch exists and fetch the file from that exact ref before editing it;
3. write back to that same branch unless the user explicitly asks for another target;
4. do not fall back to the repository default branch merely because a write tool accepts an omitted branch argument.

When a project uses a branch such as:

```text
develop-1.3.0
```

all changes intended for that unreleased version belong there. The release/default branch must remain unchanged until the project's normal PR/merge/release process intentionally promotes the work.

This rule applies equally to code, documentation, CI workflows, generated metadata and supporting integration changes in another repository. If work in plugin A requires a corresponding change in plugin B, first resolve plugin B's own active development branch; do not assume both repositories use `main` or the same branch name.

If a change was accidentally written to the wrong branch:

- restore the unintended branch to its pre-change state without carrying unrelated commits backward;
- reapply the intended change on the correct development branch;
- verify both branch heads afterward;
- document the branch mistake if the development guide was not explicit enough to prevent recurrence.

A mismatch such as `1.4.0-dev` in code while the archive is named `plugin_1.4.0_2.1.1.zip` creates unnecessary upgrade ambiguity and can cause false version differences.


For repositories supporting multiple Geeklog compatibility artifacts, use an unambiguous convention such as:

```text
plugin_1.4.0_2.1.1.zip
plugin_1.4.0_2.2.2.zip
```

when the builds are genuinely different.

Do not encode a Geeklog version in the filename if the same archive supports the complete declared Geeklog range and the project convention does not require it.

## CI should validate the archive itself

A release workflow should not merely run tests on the repository and then assume the ZIP is correct.

For Geeklog 2.1.1-compatible packages, the ZIP **must contain one top-level directory whose name is the plugin id**. The plugin files must live inside that directory.

Correct:

```text
myplugin_1.2.3_2.1.1.zip
└── myplugin/
    ├── autoinstall.php
    ├── functions.inc
    ├── public_html/
    ├── admin/
    ├── language/
    └── ...
```

Incorrect:

```text
myplugin_1.2.3_2.1.1.zip
├── autoinstall.php
├── functions.inc
├── public_html/
└── ...
```

This distinction matters because Geeklog 2.1.1 derives the plugin directory name from the archive's first top-level entry. A ZIP may therefore be structurally valid and pass `unzip -t`, yet still be unusable by Geeklog.

A typical failure mode is:

- upload reports success;
- no installation follows;
- the plugin does not appear in the list of uninstalled plugins;
- no useful error is written to the plugin log.

That usually means the archive was unpacked with the wrong top-level layout, so Geeklog never finds:

```text
plugins/PLUGIN/functions.inc
plugins/PLUGIN/autoinstall.php
```

CI should inspect the produced archive and verify:

- exactly one expected root plugin directory;
- required files present **under that root directory**;
- forbidden files absent;
- PHP syntax of shipped files;
- archive name/version consistency;
- at least one critical runtime file extracted from the ZIP is byte-for-byte identical to the source file from the commit being packaged.

For higher confidence, compare several critical files, especially files that control plugin bootstrap, installation, upgrades, and core runtime behavior, for example:

- `autoinstall.php`;
- `functions.inc`;
- `include/functions.php`;
- the plugin's main class or service file.

A release workflow must validate the **actual extracted ZIP payload**, not only the staging directory or repository tree. This catches stale archives, incorrect staging roots, omitted files, and packaging workflows that accidentally publish code from an older commit.

A practical pattern is:

```bash
rm -rf verify
mkdir verify
unzip -q "$ARCHIVE" -d verify

cmp -s "autoinstall.php" "verify/$PLUGIN/autoinstall.php"
cmp -s "functions.inc" "verify/$PLUGIN/functions.inc"
cmp -s "include/functions.php" "verify/$PLUGIN/include/functions.php"
```

If any comparison fails, the distribution job must fail. Do not publish the ZIP.

A practical shell check is:

```bash
PLUGIN="myplugin"
ARCHIVE="dist/myplugin_1.2.3_2.1.1.zip"

unzip -t "$ARCHIVE"

mapfile -t top_level_entries < <(
    unzip -Z1 "$ARCHIVE" |
        sed 's#^\./##' |
        cut -d/ -f1 |
        sed '/^$/d' |
        sort -u
)

if [ "${#top_level_entries[@]}" -ne 1 ] || [ "${top_level_entries[0]}" != "$PLUGIN" ]; then
    echo "Invalid Geeklog package layout"
    exit 1
fi

for required in \
    "$PLUGIN/autoinstall.php" \
    "$PLUGIN/functions.inc" \
    "$PLUGIN/public_html/index.php"
do
    unzip -Z1 "$ARCHIVE" | grep -Fxq "$required" || exit 1
done
```

Do not use a packaging command that changes directory into the plugin payload and then runs:

```bash
zip -r archive.zip .
```

unless the staging directory itself already contains the required `PLUGIN/` root. Otherwise the plugin files will be written directly at the ZIP root.

A safer staging pattern is:

```text
STAGE/
└── myplugin/
    └── plugin payload
```

and then:

```bash
cd "$STAGE"
zip -r "$ARCHIVE" myplugin
```

The archive is not considered release-ready until this layout check passes.

---

# 19. Minimum viable plugin checklist

A plugin should not be called operational until the relevant checks pass.

## Definition

- [ ] plugin name and version defined;
- [ ] release version is canonical and stable (for example `1.4.0`, not `1.4.0-dev`);
- [ ] runtime version, upgrade version, documentation and archive name are consistent;
- [ ] every write targeted the explicitly active development branch rather than assuming the repository default branch;
- [ ] cross-repository supporting changes used each repository's own active development branch;
- [ ] simplest robust design identified before adding compatibility branches;
- [ ] one source of truth defined for owned data and behavior;
- [ ] repeated exceptions reviewed for an underlying contract/design problem;
- [ ] Geeklog compatibility defined;
- [ ] PHP compatibility defined;
- [ ] data ownership defined;
- [ ] required tables identified;
- [ ] required permissions identified.

## Installation lifecycle

- [ ] clean install succeeds;
- [ ] activation succeeds;
- [ ] deactivation succeeds;
- [ ] reactivation succeeds;
- [ ] uninstall succeeds;
- [ ] reinstall succeeds;
- [ ] failed-install rollback tested;
- [ ] no duplicate groups/features/configuration after retry.

## Database

- [ ] tables use `$_TABLES`;
- [ ] no hardcoded `gl_` prefix;
- [ ] SQL identifiers/values handled safely;
- [ ] schema upgrade path exists.

## Configuration

- [ ] new-install defaults work;
- [ ] existing-install migration works;
- [ ] labels/options exist in supported languages;
- [ ] no user-visible strings hardcoded in PHP/templates/JavaScript;
- [ ] all referenced language keys exist in every maintained language file;
- [ ] JavaScript-visible messages are supplied by language data rather than embedded English fallbacks;
- [ ] at least one non-English public/admin/configuration pass has been tested;
- [ ] missing configuration does not cause bootstrap fatal errors;
- [ ] configuration migration is idempotent.

## Callback/bootstrap integrity

- [ ] complete source tree searched before adding/moving Plugin API callbacks;
- [ ] each important Plugin API callback has one canonical definition;
- [ ] canonical callback files are loaded on the required runtime paths;
- [ ] duplicate callback definitions are guarded by CI/source-contract tests where useful;
- [ ] plugin-owned globals used by `functions.inc` / `install_defaults.php` are explicitly declared for autoinstall scope;
- [ ] language/config/default arrays are validated before merge/use;
- [ ] real Geeklog 2.1.1 upload/autoinstall path tested, not only normal bootstrap;
- [ ] disabled plugin does not leave callable public/admin behavior unintentionally active.

## Administration

- [ ] administration entry point checks ACL;
- [ ] administration home/first page explains the plugin purpose when needed;
- [ ] concise Getting started / Help path exists for non-trivial workflows;
- [ ] administration pages are responsive and pleasant to scan/use;
- [ ] forms, lists and destructive actions are visually separated;
- [ ] state-changing actions use security tokens;
- [ ] inputs validated;
- [ ] output escaped;
- [ ] local navigation coherent;
- [ ] native administration entry exposed with `plugin_getadminoption_PLUGIN()` when applicable;
- [ ] Command & Control entry exposed with `plugin_cclabel_PLUGIN()` when applicable;
- [ ] Configuration reachable when applicable;
- [ ] substantial UI uses templates/assets.

## Public

- [ ] public page renders with `COM_createHTMLDocument()`;
- [ ] public design reviewed against semantic/responsive/accessibility guidelines;
- [ ] SEO identity/title/canonical/indexability reviewed where the page is public/indexable;
- [ ] anonymous ACL tested;
- [ ] logged-in ACL tested;
- [ ] administrator ACL tested;
- [ ] empty state tested;
- [ ] menu entry added when appropriate;
- [ ] disabled/inaccessible behavior tested.

## Assets

- [ ] CSS separated;
- [ ] JavaScript separated;
- [ ] CSS URLs are versioned;
- [ ] JavaScript URLs are versioned;
- [ ] asset version changes when the shipped file changes;
- [ ] assets loaded on `/plugin/`;
- [ ] assets loaded on `/plugin/index.php` when applicable;
- [ ] cache-busting/versioning strategy present;
- [ ] no theme-specific dependency unless optional/documented.

## Interoperability

- [ ] Item Info follows the documented return contract;
- [ ] provider tested through at least one real maintained consumer when one exists;
- [ ] FAQ association acceptance profile tested when the plugin is expected to host contextual FAQs;
- [ ] consumers invoked through shared dispatchers tolerate the previous supported/partially migrated state of their own schema without breaking provider pages;
- [ ] compatibility SQL uses centralized resolved field expressions/defaults consistently and does not later reference a missing new column directly;
- [ ] manual/contextual relationship placement, when supported, reuses the same ACL, deduplication, inheritance and conflict rules as automatic rendering;
- [ ] consumers normalize scalar/positional/associative Item Info forms where compatibility requires it;
- [ ] administration Item Info lookups use the current user's UID when appropriate;
- [ ] canonical URLs are provider-owned and resolved without hard-coded consumer routing;
- [ ] stable IDs remain unambiguous when several subtypes share one provider;
- [ ] addressable containers/root/category/album/forum resources are exposed when useful;
- [ ] provider-family role (content/navigation/relationship/service) is explicit where applicable;
- [ ] third-party private SQL is not used as a shortcut around missing contracts.

## Content lifecycle

- [ ] create works;
- [ ] edit works;
- [ ] delete works;
- [ ] lifecycle events emitted where applicable;
- [ ] search/What's New/Item Info/etc. tested where implemented.

## Persistent storage

- [ ] persistent files do not live in cache;
- [ ] path is derived from active site context;
- [ ] unauthorized direct download is prevented where necessary;
- [ ] multisite isolation verified.

## Upgrade

- [ ] previous released version can upgrade;
- [ ] schema migration tested;
- [ ] configuration migration tested;
- [ ] interrupted/repeated upgrade behavior tested;
- [ ] shared-files compatibility considered;
- [ ] new runtime code does not require newly added owned-schema columns before the site's upgrade has completed.

## Compatibility

- [ ] PHP 5.6 syntax/runtime tested where still supported;
- [ ] PHP 8.1 syntax/runtime tested;
- [ ] Geeklog 2.1.1 runtime tested where supported;
- [ ] Geeklog 2.2.2 runtime tested;
- [ ] no unsupported syntax slipped into shipped files.

## Packaging

- [ ] installable ZIP produced;
- [ ] root folder correct;
- [ ] dotfiles/development metadata excluded;
- [ ] archive contents validated by CI;
- [ ] archive install tested.

---

# 20. Frequent mistakes observed during modernization

These mistakes are especially expensive because they often produce a plugin that looks almost complete while one lifecycle step remains broken.

## Localization and visible-string discipline

User-visible text must not be hardcoded in PHP, templates, or JavaScript.

This applies to all plugin surfaces, including:

- public pages;
- administration pages and navigation;
- buttons and form labels;
- headings and table columns;
- validation and status messages;
- AJAX responses displayed to users;
- JavaScript loading, error and fallback messages;
- accessibility text such as `aria-label`;
- Configuration labels, select values, fieldset/tab names and tooltips.

Prefer plugin language arrays as the single source of visible strings. Keep related surfaces separated when useful, for example:

```php
$LANG_PLUGIN_COMMON
$LANG_PLUGIN_ADMIN
$LANG_PLUGIN_RELATIONS
$LANG_PLUGIN_COVERAGE
```

Do not keep an English literal in code as a silent fallback for a missing language key. If a supported language does not yet have a native translation for a new key, keep the key contract complete in that language file and use an explicit documented fallback there instead. This avoids hidden English strings spread through application code and prevents `Undefined array key` warnings.

For JavaScript, do not create a second untracked translation system. Pass localized strings from PHP to the script through rendered data, JSON or an equivalent provider-owned mechanism, then let JavaScript consume those values.

Example:

```php
<select
  data-msg-loading="<?= htmlspecialchars($LANG_PLUGIN_ADMIN['loading']) ?>"
  data-msg-error="<?= htmlspecialchars($LANG_PLUGIN_ADMIN['load_error']) ?>">
```

The corresponding JavaScript should consume these values instead of embedding English text.

### Language parity

Every maintained language file should expose the same functional key contract for the current plugin version.

Before release:

1. collect the language keys actually referenced by PHP/templates/JavaScript;
2. verify that every referenced key exists in the primary language;
3. verify that every supported language defines the same required keys;
4. scan templates and generated HTML for visible hardcoded strings;
5. scan JavaScript for visible fallback strings that bypass language files;
6. test at least one non-English language on both public and administration surfaces.

Do not assume that adding one translated file is sufficient. Existing historical language files must also be kept structurally current when new public/admin/configuration keys are introduced.

A practical maintenance rule is:

> **Code consumes language keys; language files own visible wording.**

## Hardcoded database prefix

```text
❌ hardcode gl_mytable
✅ register and use $_TABLES
```

## New configuration without upgrade

```text
❌ add an option only to install_defaults.php
✅ bump version when required + migrate existing installations idempotently
```

## Asset detection only for index.php

```text
❌ load JS only for /plugin/index.php
✅ support /plugin/ and /plugin/index.php where both routes are valid
```

## Wrong document/footer mechanism

```text
❌ assume 'footercode' in COM_createHTMLDocument() replaces Plugin API asset hooks
✅ use the appropriate Geeklog script/CSS APIs or plugin_getfootercode_PLUGIN()
```

## Duplicate Plugin API callbacks

```text
❌ add plugin_getadminoption_PLUGIN() or another callback to a new file without searching the old code
✅ search the complete tree, keep one canonical definition, and guard it in CI when appropriate
```

## Wrong Item Info permission context

```text
❌ use uid=0 in administration and conclude that a provider has no title/content
✅ use the current authenticated UID for administration lookups unless public visibility is intentionally being audited
```

## Hard-coded provider routing in a consumer

```text
❌ teach FAQ/Hub/Agent how Documents, Maps, Videos, etc. build URLs
✅ ask the provider through plugin_idtourl_*() and/or Item Info url/title fields
```

## Ambiguous IDs for multi-subtype providers

```text
❌ reuse numeric id "8" for both forum 8 and category 8 when APIs may carry only type + id
✅ expose stable provider-owned IDs such as forum:8 and category:8, plus subtype when supported
```

## Wrong callback data shape

```text
❌ return a convenient custom structure from a Geeklog callback
✅ follow the exact callback contract documented for the supported Geeklog versions
```

## Persistent uploads in cache

```text
❌ store uploads under cache/
✅ use persistent, site-aware storage derived from path_data
```

## Admin page protected only by navigation

```text
❌ hide the menu item and assume the page is protected
✅ enforce ACL at every entry point and mutation
```

## State change without CSRF protection

```text
❌ accept delete/update actions directly from request parameters
✅ require and validate Geeklog security tokens
```

## Theme coupling

```text
❌ build core plugin behavior specifically for Eclipse or Denim
✅ produce Geeklog-compatible output and keep theme-specific integration optional
```

## Bootstrap-heavy functions.inc

```text
❌ load large optional subsystems unconditionally from functions.inc
✅ keep bootstrap minimal and load feature code only where needed
```

## Assuming top-level scope during autoinstall

```text
❌ assume $_PLUGIN_DEFAULT / language arrays are global because normal bootstrap works
❌ rely on require_once to make included variables globally visible

✅ declare plugin-owned globals explicitly
✅ validate arrays before array_merge() or indexed access
✅ test the actual Geeklog 2.1.1 upload/autoinstall path
```

Geeklog 2.1.1 can include plugin files from inside `plugin_do_autoinstall()`, so those files inherit function scope. A plugin may therefore work during normal requests but fail only while being installed from an uploaded archive.

## Upgrade assumes all multisite databases are synchronized

```text
❌ deploy shared files that require the newest schema immediately
✅ keep new files compatible with the previous supported persisted state until each site upgrades
```

## Publishing an unchecked archive

```text
❌ trust the repository build because tests passed before ZIP creation
✅ inspect and validate the actual generated archive in CI
```

## Shipping development suffixes as the release version

```text
❌ pi_version = 1.4.0-dev
❌ archive says 1.4.0 while runtime reports 1.4.0-dev

✅ pi_version = 1.4.0
✅ keep development status in branch/roadmap/release notes
```

The canonical runtime version should describe the release identity, not the development state of the branch.

## Packaging files directly at the ZIP root

```text
❌ archive.zip/autoinstall.php
❌ archive.zip/functions.inc

✅ archive.zip/PLUGIN/autoinstall.php
✅ archive.zip/PLUGIN/functions.inc
```

Geeklog 2.1.1 derives the plugin name from the first top-level archive entry. A flat ZIP can therefore report a successful upload while never becoming installable.

## Consumer extension callback assumes its newest schema

```text
❌ PLG_itemDisplay() callback unconditionally queries newly added relation columns before every supported installation has migrated
❌ fix each affected provider with a provider-specific bypass

✅ keep schema compatibility inside the consuming plugin
✅ detect the owned schema once through a centralized helper
✅ use safe defaults/resolved SQL expressions during the supported transition
✅ let the normal upgrade converge the schema to the current version
```

A correctly implemented provider should not need to know whether FAQ, Hub, or another consumer has finished migrating its private relation tables.

## Fixing symptoms with compatibility patches everywhere

```text
❌ accumulate route-, version-, provider-, page-, or theme-specific patches without a clear contract
✅ identify the shared lifecycle/API/data-model cause, fix it once, and isolate only the unavoidable compatibility code
```

If several exceptions keep appearing around the same behavior, do not add a fourth exception automatically. Re-evaluate whether the shared abstraction, identity model, provider contract, ownership boundary, or lifecycle is wrong.

A local patch is acceptable only when:

- the incompatibility is real and bounded;
- the generic/shared contract cannot safely represent the difference;
- the patch is isolated;
- the reason is documented;
- tests cover the compatibility branch;
- removal conditions are understood.


---

# 21. Recommended implementation order

For a new plugin, the following order catches architectural mistakes early:

```text
1. define plugin contract and ownership boundaries
2. choose the simplest robust shared design and identify unavoidable compatibility boundaries
4. create minimal file structure
3. define table/data ownership
5. implement install + uninstall
6. implement configuration defaults
7. install on a clean Geeklog instance
8. build protected administration
9. build one working public page
10. add CRUD
11. add configuration migrations
12. add assets
13. add public/admin navigation
14. add optional blocks
15. add Search / What's New / Item Info / sitemap / feeds as relevant
16. add persistent file handling if required
17. test ACL and security tokens
18. test upgrade from previous version
19. test supported Geeklog/PHP combinations
20. build the release ZIP
21. validate the ZIP itself
```

This sequence is intentionally lifecycle-first.

A plugin should become installable, removable, administrable, publicly renderable, and upgradeable **before** optional integrations make it larger.

---

# 22. Memorandum compliance baseline

A modernized plugin should not only work functionally; it should also conform to the development expectations documented in this repository.

Before release, verify the plugin against the relevant Memorandum documents.

At minimum, review:

- Plugin API compatibility;
- configuration migration;
- administration navigation;
- persistent storage;
- multisite isolation;
- shared-files upgrade safety;
- content interoperability where applicable;
- capability declaration where applicable;
- metadata manifest expectations;
- CSS/JS asset versioning;
- release packaging and upgrade behavior.

The plugin's own README or roadmap should explicitly identify any Memorandum recommendation that is intentionally not implemented and explain why.

A plugin should not claim Memorandum alignment merely because its main feature works.

---

# 23. When to read the specialized documents

Use this guide as the starting point, then move to the detailed references when implementing each subsystem.

| Need | Reference |
| --- | --- |
| Existing Plugin API callbacks | [plugin-api-reference-2.2.2.md](plugin-api-reference-2.2.2.md) |
| Configuration and existing-install migration | [plugin-configuration-migration-guide-2.2.2.md](plugin-configuration-migration-guide-2.2.2.md) |
| Configuration contextual help/tooltips | [plugin-configuration-tooltips.md](plugin-configuration-tooltips.md) |
| Admin menus and local section navigation | [plugin-admin-navigation.md](plugin-admin-navigation.md) |
| Administration usability and first-use help | [plugin-admin-ux-guidelines.md](plugin-admin-ux-guidelines.md) |
| CSS/JS loading, cache-busting and packaged assets | [plugin-asset-loading-versioning.md](plugin-asset-loading-versioning.md) |
| Public design, responsive behavior and accessibility | [plugin-public-design-guidelines.md](plugin-public-design-guidelines.md) |
| Public page SEO baseline | [plugin-seo-public-page-guidelines.md](plugin-seo-public-page-guidelines.md) |
| Persistent files | [plugin-persistent-storage-guide.md](plugin-persistent-storage-guide.md) |
| Multisite constraints | [multisite-development-principles.md](multisite-development-principles.md) |
| Shared files with independently upgraded sites | [plugin-shared-files-upgrade-safety.md](plugin-shared-files-upgrade-safety.md) |
| Structured content interoperability | [plugin-content-interoperability-contract.md](plugin-content-interoperability-contract.md) |
| Capability discovery/services/dashboard contracts | [plugin-capability-contract.md](plugin-capability-contract.md) |
| Plugin metadata manifest | [plugin-metadata-manifest.md](plugin-metadata-manifest.md) |

---

# Final principle

The Plugin API reference answers:

> **What can a Geeklog plugin implement?**

This guide answers:

> **What must I build, and in what order, so the plugin actually works as a complete Geeklog component?**

A plugin is not finished when its main feature works.

It is finished when its complete lifecycle works:

```text
install
  ↓
activate / deactivate / reactivate
  ↓
configure
  ↓
administer
  ↓
use publicly
  ↓
secure
  ↓
integrate
  ↓
upgrade
  ↓
package
  ↓
reinstall / uninstall
```

That lifecycle should be the default development model for every modernized Geeklog plugin.
