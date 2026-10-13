# Geeklog Development & Modernization Memorandum

![Geeklog Memorandum](docs/images/geeklog-cms-memorandum-2030.png)

A practical reference for modernizing Geeklog plugins and themes today while preparing a cleaner architecture for the future.

## Shared IA architecture — decision of 13 October 2026

- **Agent** remains the provider-neutral machine access layer: authorized external reading and future actions. It is not the AI model invocation engine or a workflow scheduler.
- **IA (proposed independent plugin)** receives generation requests from Geeklog plugins (including Automation and Forum), connects to different AI providers, normalizes text/image/structured responses, tracks usage, errors and quotas, and returns results to its caller.
- **Automation** orchestrates steps, conditions, scheduling, approvals and retries of full workflows. Automation must remain useful without IA for non-AI tasks.
- **Hub** owns editorial relationships and context; Core and feature plugins own their data, permissions and business actions.
- Before implementing IA, **audit existing LLM-related code and documentation inside the [Agent repository](https://github.com/hostellerie/agent)** to identify what can be reused or extracted safely. There is **no separate LLM repository**. Do not remove working Agent features or migrate code without tests.
- Agree on shared service/capability contracts, site isolation, credentials, permission checks, sync/async job boundaries and two consumer tests (Automation and Forum). Avoid competing provider integrations and duplicate queues.
- This design is a proposal in documentation, not an implemented or released IA plugin.

## Current compatibility priority

**Effective baseline for newly maintained Geeklog plugins and themes (October 2026):**

- **Geeklog 2.2.2 minimum.** Geeklog 2.1.1 is no longer a supported runtime target.
- **PHP 5.6.40 minimum.** Maintain PHP 5.6-compatible syntax and test with PHP 5.6 and PHP 8.1/8.3 when supported by the project.
- Preserve upgrades from older installed plugin data; removing support for older Geeklog runtimes must not discard historical persisted state.
- Multisite deployments with shared files must support staggered plugin-data upgrades across sites running Geeklog 2.2.2.
- Cookie Consent does not manage Google AdSense. Google's separate consent/ad tooling stays independently configured; never imply that loading AdSense alone ensures compliant consent.

See [current runtime support policy](geeklog-runtime-support-policy.md). Historical plans and completed migration notes below may still describe previous compatibility goals; they are not the current maintenance baseline.

## How to read this repository

**Starting a new Geeklog plugin? Begin with [`plugin-development-guide.md`](plugin-development-guide.md).** It provides the end-to-end build order from plugin definition, tables and installation through configuration, administration, public pages, upgrades, packaging and common failure modes.

The documents are separated conceptually into four layers.

### 1. Current Geeklog facts

These documents describe APIs and behavior that exist in Geeklog itself.

- [`plugin-api-reference-2.2.2.md`](plugin-api-reference-2.2.2.md) — reference to the Geeklog 2.2.2 Plugin API.
- [`plugin-configuration-migration-guide-2.2.2.md`](plugin-configuration-migration-guide-2.2.2.md) — configuration and upgrade lessons observed while modernizing plugins for Geeklog 2.2.2.
- [`geeklog-2.1.1-to-2.2.2-multisite-upgrade-lessons.md`](geeklog-2.1.1-to-2.2.2-multisite-upgrade-lessons.md) — historical field guide from a 15-site shared-Core migration: backups, schema/version mismatches, optional plugins, partial failures, recovery and post-upgrade verification.
- [`geeklog-installer-upgrade-rescue-architecture.md`](geeklog-installer-upgrade-rescue-architecture.md) — proposed division of responsibilities between Monitor, Migrator SQL recovery, and native Geeklog Install/Upgrade/Rescue.

These references describe the newer end of the supported range. When maintaining compatibility with Geeklog 2.1.1, plugins and themes must verify that an API or theme capability exists before relying on it or provide a compatible fallback.

### 2. Development conventions

These are recommended engineering rules for modernized plugins and themes. They are not automatically official Geeklog requirements.

Core principles:

- initialize variables explicitly;
- keep code compatible with PHP 5.6 where current project policy requires it;
- remove PHP constructs that fail on PHP 8.x;
- use Geeklog APIs instead of direct platform bypasses where practical;
- validate input and escape output according to context;
- protect state-changing operations with Geeklog security tokens;
- preserve ACL checks on administration and sensitive actions;
- separate PHP logic from templates where practical;
- make installation and upgrades repeatable and non-destructive;
- treat persistent project data differently from disposable cache data;
- design storage and configuration with multisite isolation in mind.

Additional conventions:

- [`plugin-admin-navigation.md`](plugin-admin-navigation.md) — native Geeklog admin-menu usage and the shared fallback contract for persistent plugin-local section navigation.
- [`plugin-admin-ux-guidelines.md`](plugin-admin-ux-guidelines.md) — administration usability, responsive layout, clear workflows, first-use orientation and concise built-in help.
- [`plugin-asset-loading-versioning.md`](plugin-asset-loading-versioning.md) — external CSS/JS, contextual loading, deterministic cache-busting and packaging checks.
- [`plugin-public-design-guidelines.md`](plugin-public-design-guidelines.md) — theme-neutral public design, responsive behavior, accessibility, readable content and robust UI states.
- [`plugin-seo-public-page-guidelines.md`](plugin-seo-public-page-guidelines.md) — page-level SEO baseline for canonical public plugin resources, metadata, semantic HTML, structured data, containers and sitemap participation.
- [`plugin-persistent-storage-guide.md`](plugin-persistent-storage-guide.md)
- [`multisite-development-principles.md`](multisite-development-principles.md)
- [`plugin-shared-files-upgrade-safety.md`](plugin-shared-files-upgrade-safety.md) — required compatibility behavior when several sites share plugin files but upgrade their persisted state at different times.

### 3. Active modernization roadmap

[`geeklog-modernization-roadmap.md`](geeklog-modernization-roadmap.md) describes the projects that currently deserve implementation attention.

The present order is:

1. Menu 1.3.0
2. Documents
3. Maps
4. Eclipse
5. Store
6. Videos
7. Multisite Manager

The **Plugin Toolkit is on hold**.

Eclipse follows the same transition objective as the modernized plugins: native support for Geeklog 2.2.2 and later and PHP 5.6.40 through 8.1 and later supported versions. This broader compatibility is intended to make Eclipse adoptable by existing sites before they complete a Geeklog or PHP migration.

### 4. Future architecture

These documents describe direction and proposed contracts. They are not claims about APIs already present in Geeklog.

- [`plugin-content-interoperability-contract.md`](plugin-content-interoperability-contract.md) — recommended common contract for exposing plugin content to Hello, Hub, IndexNow, Sitemap, search, recommendations and future consumers while keeping plugins independent from each other's SQL and internals.
- [`plugin-capability-contract.md`](plugin-capability-contract.md) — shared provider-neutral capability contract for declaring reusable content, service, dashboard, navigation, relationship and diagnostic capabilities consumed by Agent, Hub, Eclipse, AdSense and future integrations.
- [`plugin-source-field-audit-mutation-contract.md`](plugin-source-field-audit-mutation-contract.md) — shared contract for permission-aware inspection of exact source text fields and controlled provider-owned mutations, initially motivated by AdSense historical-autotag audit/migration.
- [`geeklog-chatgpt-connector.md`](geeklog-chatgpt-connector.md) — architectural concept for a secure Geeklog API plugin plus a ChatGPT connector, allowing ChatGPT to discover, read and later perform explicitly authorized actions on Geeklog without modifying the Core.
- [`geeklog-2030-roadmap.md`](geeklog-2030-roadmap.md)
- [`geeklog-marketing-roadmap-2027-2030.md`](geeklog-marketing-roadmap-2027-2030.md)

For modernized content plugins such as **Maps, Documents, Videos and Store**, the interoperability baseline should now be considered during implementation, not postponed until Hub is complete. The recommended baseline is structured Item Info exposure, collection retrieval where appropriate, lifecycle event emission, and URL resolution with compatibility fallbacks across Geeklog 2.2.2+.

The long-term principle is:

> **Structure → Expose → Connect → Discover → Recommend → Automate → Act**

The goal is not to build every future feature now. The goal is to avoid architectural choices today that block interoperability later.

---

# Compatibility engineering rules

## Write for PHP 5.6, test through PHP 8.1

For projects keeping the current compatibility policy:

- do not use scalar type declarations, return type declarations, null coalescing (`??`), arrow functions, typed properties, union types, attributes, constructor property promotion, `match`, or other syntax unavailable in PHP 5.6;
- replace removed legacy constructs such as `each()` and curly-brace string/array offsets;
- initialize variables and arrays before use;
- normalize values before passing them to native functions whose PHP 8 behavior is stricter;
- avoid assumptions that only work because older PHP versions silently tolerated invalid input.

Compatibility is not achieved by writing old-style code. It is achieved by using the common safe subset of PHP 5.6.40–8.1+ and testing both ends of the range.

## Geeklog 2.2.2 and later

A modernized project should not assume every Geeklog 2.2.2 facility exists in 2.1.1.

When a newer API or theme capability materially improves integration:

1. use it when available;
2. provide a safe fallback for older supported Geeklog versions;
3. isolate compatibility code so it can later be removed cleanly;
4. document when a feature behaves differently across versions.

Plugins should avoid theme-specific assumptions in business logic. They should produce theme-compatible output through Geeklog conventions rather than depend on Denim, Eclipse, or another specific theme unless the integration is explicitly optional.

Eclipse itself should provide version-aware compatibility layers where Geeklog 2.2.2 introduces theme capabilities that do not exist in Geeklog 2.1.1.

## Rendering

For plugins targeting the current transition baseline **Geeklog 2.2.2 and later**, use `COM_createHTMLDocument()` for complete-page rendering. It is available in the supported range and is the common rendering path across these versions.

Do not build or retain modernized plugin pages around `COM_siteHeader()` / `COM_siteFooter()`. These legacy functions are not available in Geeklog 2.2.x and can therefore produce fatal errors or completely blank pages when a plugin that still uses them is opened under Geeklog 2.2.2.

When a page works under Geeklog 2.1.1 but becomes blank under Geeklog 2.2.2, audit the rendering path immediately for direct or indirect calls to `COM_siteHeader()` and `COM_siteFooter()` before investigating the plugin's `.thtml` files.

A modernized page should build its content first and render the final document once, for example through `COM_createHTMLDocument($content, $options)`. Keep page content generation separate from document-shell rendering so the same business logic remains compatible with different Geeklog themes.

Do not describe other legacy rendering facilities as removed unless that statement has been verified for the exact Geeklog version being targeted.

## Assets

Plugin-owned static CSS and JavaScript should be stored in external files, loaded only in the contexts that need them where practical, and served with a deterministic cache-busting version. Administration assets follow the same rule as public assets.

For header assets, use Geeklog's normal Plugin API integration such as `plugin_getheadercode_PLUGIN()` where it is compatible with the project's supported Geeklog range. Avoid large reusable `<style>` or `<script>` blocks embedded directly in templates.

At minimum, asset URLs should include the plugin release version (for example `admin.css?v=1.2.0`). A plugin version plus file modification timestamp is also acceptable. The version must change when shipped asset content changes; do not use random values that disable caching on every request.

Installable archives must contain every referenced CSS/JS asset, and packaging CI should verify important asset paths. A normal plugin upgrade should not require the administrator to manually clear the browser cache.

See [Plugin asset loading and cache versioning](plugin-asset-loading-versioning.md).

Avoid embedding unescaped PHP values directly into JavaScript. For structured PHP-to-JavaScript data, JSON encoding with appropriate escaping is preferred.

Eclipse and plugins should account for differences in asset handling between Geeklog 2.2.2 instead of assuming only the newer path.

## Database access

Use Geeklog's `DB_*` abstraction for plugin database access. Validate or cast identifiers and escape string values used in SQL.

Do not assume that SQL escaping replaces application-level validation.

## Security

State-changing operations should use Geeklog's CSRF token mechanisms where supported by the target versions.

Administrative and sensitive operations must enforce Geeklog ACL permissions independently of whether a link or button is visible in the interface.

Input filtering and output escaping are separate responsibilities. Escape values for the output context in which they are rendered.

## Templates

Prefer `.thtml` templates for significant presentation markup. Keep business logic in PHP and presentation in templates where practical.

Plugin output should remain theme-independent. Eclipse may serve as a modern reference implementation, but plugins should not require Eclipse unless explicitly documented.

Eclipse should preserve native compatibility with both the legacy theme expectations of Geeklog 2.1.1 and the newer theme architecture of Geeklog 2.2.2 through explicit compatibility handling rather than separate incompatible editions where practical.

## Blank pages and silent fatal errors

A completely blank page should be treated as a bootstrap/runtime failure until proven otherwise, not as a template problem.

When a plugin page renders no Geeklog header, no Root Debugging message and no useful HTML, diagnose the common execution path first:

- verify the plugin bootstrap files loaded by `lib-common.php`, especially `functions.inc` and any files it requires unconditionally;
- run `php -l` on every `.php` and `.inc` file included in the distribution, because PHP 8 may reject syntax that older PHP versions accepted;
- inspect the exact server `error.log` entry produced by the request; do not rely only on the browser response or Geeklog Root Debugging;
- search for unconditional `die()`, `exit`, direct-inclusion guards and early returns in files loaded globally;
- distinguish page-specific failures from bootstrap failures by testing both an administrative page and a public page that share the same plugin loader;
- audit direct array access to optional request, session, configuration and legacy serialized-data keys, since PHP 8 reports undefined offsets/keys that older versions often tolerated;
- ensure new-install configuration contains every key used during unconditional plugin bootstrap;
- validate that language arrays required during bootstrap are loaded before use, or provide bootstrap-safe defaults;
- avoid suppressing the root cause with `@`; suppression may hide the only useful diagnostic on production-like hosts.

A plugin distribution workflow should lint all shipped `.php` and `.inc` files before producing or publishing an installable archive. A build containing a PHP syntax error must fail before the archive reaches `dist/`.

For a blank page affecting several unrelated plugin URLs, start with the shared bootstrap path rather than the page templates. Templates should only be investigated after the common bootstrap, syntax and runtime checks pass.

## Installation and upgrades

Modernization must preserve existing installations.

Upgrade routines should be:

- sequential;
- repeatable where practical;
- non-destructive;
- safe when interrupted;
- careful not to delete legacy data until migration has been verified.

Modernized plugins should also implement a complete `plugin_autouninstall_PLUGIN()` contract. This is required not only for a normal uninstall, but also so Geeklog can roll back a failed installation cleanly. The auto-uninstall metadata should list every plugin-owned table, group, feature, PHP block and plugin variable that Geeklog must remove. It must be available during the install path as well as the normal plugin lifecycle when required by the target Geeklog versions. A failed installation must not leave groups, features, configuration records or tables that cause the next installation attempt to fail with duplicate-entry errors.

Installation tests should therefore include an intentional failure after groups/features/tables have begun to be created, followed by verification that `PLG_uninstall()` removes the partial state and that a second install succeeds without manual database cleanup.

Configuration defaults for new installations and configuration migration for existing installations are separate concerns and should be tested separately.

When plugin files can be shared by several Geeklog sites, new files must also remain operational with the previous supported persisted plugin state until the active site's explicit upgrade completes. See [`plugin-shared-files-upgrade-safety.md`](plugin-shared-files-upgrade-safety.md).

## Persistent storage

User files and persistent plugin or theme data must not be stored in locations that Geeklog treats as disposable cache.

A project should derive persistent paths from the current site's configuration so that multisite installations remain isolated. See [`plugin-persistent-storage-guide.md`](plugin-persistent-storage-guide.md).

## Multisite

Multisite is a development constraint, not merely a future plugin feature.

Every project storing persistent data should define:

- current site context;
- data isolation;
- configuration isolation;
- permissions context;
- migration behavior.

The future **Multisite Manager** can remain a later implementation project while these principles apply immediately. See [`multisite-development-principles.md`](multisite-development-principles.md).

For shared plugin directories, upgrading one site's persisted state must not be required for other sites to keep running after the new files are deployed. The shared-files transition policy is defined in [`plugin-shared-files-upgrade-safety.md`](plugin-shared-files-upgrade-safety.md).

## Plugin content interoperability

Modernized content plugins should expose content through Geeklog APIs instead of requiring consumers to query plugin tables directly.

The recommended baseline is:

- `plugin_getiteminfo_PLUGIN()` for structured metadata;
- collection support using `'*'` where the plugin can expose multiple items;
- common collection options such as `since`, `limit`, and `order` where applicable;
- `PLG_itemSaved()` on successful creations and updates;
- `PLG_itemDeleted()` on successful deletions;
- `plugin_idtourl_PLUGIN()` where supported, with Item Info URL fallback for older Geeklog versions;
- a full-item public rendering extension point through `PLG_itemDisplay($id, $type)` so other plugins can contribute contextual server-rendered fragments without provider-specific coupling.

This baseline is intended to make the same plugin content reusable by Hello, Hub, IndexNow, Sitemap and future consumers without introducing a separate API for each integration.

See [`plugin-content-interoperability-contract.md`](plugin-content-interoperability-contract.md) for the content contract and [`plugin-capability-contract.md`](plugin-capability-contract.md) for shared capability discovery, specialized services and the `dashboard.summary` convention used by generic consumers such as Eclipse.

## Shared capability interoperability

Modernized plugins should declare reusable capabilities once rather than forcing Agent, Hub, Eclipse, AdSense or another consumer to maintain a plugin-specific registry. Capability declaration is descriptive and must map to existing Geeklog APIs, bounded `PLG_invokeService()` services or documented versioned provider contracts.

Eclipse acts as a presentation consumer: Eclipse 1.2 now discovers active plugin capabilities and renders provider-owned `dashboard.summary` data instead of querying plugin-private tables. Structured provider metrics take precedence over legacy plugin statistics for the same provider, explicit alerts plus conventional `pending` / `drafts` metrics can surface in **Needs attention**, and legacy Geeklog statistics remain a fallback for older plugins. Hub should consume the same declarations and also expose its own relationship/context capabilities. Agent should adapt the same provider contracts to machine-readable resources and tools without becoming a required dependency for plugin-to-plugin interoperability.

See [`plugin-capability-contract.md`](plugin-capability-contract.md).

---

# API terminology

Four concepts must remain distinct.

### Geeklog Plugin API

The existing `plugin_<api>_<plugin>()` hooks used by Geeklog and plugins.

### Plugin Content Interoperability Contract

A recommended harmonization layer built primarily on existing Geeklog Plugin APIs such as Item Info, lifecycle events and URL resolution. Its purpose is to make content plugins reusable by multiple consumers without coupling those consumers to plugin internals.

### Plugin Capability Contract

A shared provider-neutral declaration layer for capabilities that can be reused by Agent, Hub, Eclipse, AdSense and other consumers. It does not replace Geeklog APIs: it identifies available resources/services/actions and maps them to Item Info, search, lifecycle, `PLG_invokeService()` or another documented provider-owned contract.

### Future Data API

A proposed structured interface for exposing content, products, services and other entities to applications and external consumers.

### Future Events API

A proposed common event contract allowing plugins to publish meaningful events without depending directly on Marketing, Analytics, Notifications or another consumer.

Future Marketing functionality should consume common events rather than become Geeklog's universal event bus.

---

# Project status terminology

To avoid confusing ideas with active development, roadmap documents should use these meanings:

- **Active** — implementation or stabilization currently underway.
- **Planned** — accepted next-stage work, but not the current focus.
- **Architectural concept** — useful future direction whose implementation has not started.
- **On hold** — intentionally not receiving current development effort.

Services, Booking, Entity/Knowledge, Tools, Recommendations and AI/Agents are currently architectural concepts unless their status is explicitly changed in their own repositories.

---

# Long-term direction

Geeklog already provides a mature publishing foundation. The next architectural opportunity is to make its information easier to structure, expose and connect while keeping plugins focused and independent.

Near-term modernization comes first. Future architecture should guide today's interfaces without forcing unfinished abstractions into projects that still need stabilization.
