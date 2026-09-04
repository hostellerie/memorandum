# Geeklog Development & Modernization Memorandum

![Geeklog Memorandum](docs/images/geeklog-cms-memorandum-2030.png)

A practical reference for modernizing Geeklog plugins and themes today while preparing a cleaner architecture for the future.

## Current compatibility priority

For **plugins and the Eclipse theme currently being modernized**, the working compatibility target is:

- **Geeklog 2.1.1 through 2.2.2**
- **PHP 5.6 through PHP 8.1**

This compatibility range is intentional. It allows existing Geeklog sites to adopt modernized plugins and Eclipse before or during a staged migration to Geeklog 2.2.2 and newer PHP versions.

New code should therefore avoid introducing syntax or runtime requirements that prevent PHP 5.6 compatibility unless a project explicitly changes its own support policy.

The longer-term architecture described in this repository may target Geeklog 2.2.2 and PHP 8.1+ once the migration period is complete. Future targets must not be confused with the compatibility requirements of projects being modernized today.

---

## How to read this repository

The documents are separated conceptually into four layers.

### 1. Current Geeklog facts

These documents describe APIs and behavior that exist in Geeklog itself.

- [`plugin-api-reference-2.2.2.md`](plugin-api-reference-2.2.2.md) — reference to the Geeklog 2.2.2 Plugin API.
- [`plugin-configuration-migration-guide-2.2.2.md`](plugin-configuration-migration-guide-2.2.2.md) — configuration and upgrade lessons observed while modernizing plugins for Geeklog 2.2.2.

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

Eclipse follows the same transition objective as the modernized plugins: native support for Geeklog 2.1.1 through 2.2.2 and PHP 5.6 through 8.1. This broader compatibility is intended to make Eclipse adoptable by existing sites before they complete a Geeklog or PHP migration.

### 4. Future architecture

These documents describe direction and proposed contracts. They are not claims about APIs already present in Geeklog.

- [`plugin-content-interoperability-contract.md`](plugin-content-interoperability-contract.md) — recommended common contract for exposing plugin content to Hello, Hub, IndexNow, Sitemap, search, recommendations and future consumers while keeping plugins independent from each other's SQL and internals.
- [`geeklog-2030-roadmap.md`](geeklog-2030-roadmap.md)
- [`geeklog-marketing-roadmap-2027-2030.md`](geeklog-marketing-roadmap-2027-2030.md)

For modernized content plugins such as **Maps, Documents, Videos and Store**, the interoperability baseline should now be considered during implementation, not postponed until Hub is complete. The recommended baseline is structured Item Info exposure, collection retrieval where appropriate, lifecycle event emission, and URL resolution with compatibility fallbacks across Geeklog 2.1.1–2.2.2.

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

Compatibility is not achieved by writing old-style code. It is achieved by using the common safe subset of PHP 5.6–8.1 and testing both ends of the range.

## Geeklog 2.1.1 through 2.2.2

A modernized project should not assume every Geeklog 2.2.2 facility exists in 2.1.1.

When a newer API or theme capability materially improves integration:

1. use it when available;
2. provide a safe fallback for older supported Geeklog versions;
3. isolate compatibility code so it can later be removed cleanly;
4. document when a feature behaves differently across versions.

Plugins should avoid theme-specific assumptions in business logic. They should produce theme-compatible output through Geeklog conventions rather than depend on Denim, Eclipse, or another specific theme unless the integration is explicitly optional.

Eclipse itself should provide version-aware compatibility layers where Geeklog 2.2.2 introduces theme capabilities that do not exist in Geeklog 2.1.1.

## Rendering

For Geeklog versions supporting the modern document rendering path, prefer `COM_createHTMLDocument()` for new or substantially modernized pages.

Do not describe legacy rendering functions as universally removed unless that statement has been verified for the exact Geeklog version being targeted. Compatibility code may still be necessary for older supported releases.

## Assets

Use Geeklog's script and CSS management APIs where they are available and compatible with the supported Geeklog range. Register assets before final document rendering.

Avoid embedding unescaped PHP values directly into JavaScript. For structured PHP-to-JavaScript data, JSON encoding with appropriate escaping is preferred.

Eclipse and plugins should account for differences in asset handling between Geeklog 2.1.1 and 2.2.2 instead of assuming only the newer path.

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

## Installation and upgrades

Modernization must preserve existing installations.

Upgrade routines should be:

- sequential;
- repeatable where practical;
- non-destructive;
- safe when interrupted;
- careful not to delete legacy data until migration has been verified.

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
- `plugin_idtourl_PLUGIN()` where supported, with Item Info URL fallback for older Geeklog versions.

This baseline is intended to make the same plugin content reusable by Hello, Hub, IndexNow, Sitemap and future consumers without introducing a separate API for each integration.

See [`plugin-content-interoperability-contract.md`](plugin-content-interoperability-contract.md) for the full proposed contract and implementation priorities.

---

# API terminology

Four concepts must remain distinct.

### Geeklog Plugin API

The existing `plugin_<api>_<plugin>()` hooks used by Geeklog and plugins.

### Plugin Content Interoperability Contract

A recommended harmonization layer built primarily on existing Geeklog Plugin APIs such as Item Info, lifecycle events and URL resolution. Its purpose is to make content plugins reusable by multiple consumers without coupling those consumers to plugin internals.

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
