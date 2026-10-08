# Geeklog Plugin Content Interoperability Contract

## Purpose

This document defines a practical interoperability target for Geeklog content plugins such as **Maps, Documents, Videos, Store, Calendar, Links, Polls, Static Pages**, and similar extensions.

The objective is to let one plugin consume another plugin's content without knowing its SQL tables, column names, internal paths, or business logic.

This contract is intended to support several consumers over time:

- **Hello** — retrieve new or recently updated content for notifications and digests;
- **Hub** — identify content, maintain relations, build content hubs, and react to lifecycle changes;
- **IndexNow** — resolve URLs that should be submitted after changes;
- **Sitemap** — discover addressable content;
- **Content Syndication** — expose plugin content through Geeklog-managed RSS/Atom feeds;
- **Statistics** — let plugins contribute to Geeklog's global statistics views;
- **Dashboards and reporting** — retrieve structured popular-content data without plugin-specific SQL;
- **Search and recommendation features** — reuse structured metadata instead of plugin-specific SQL.

The recommended approach is to build on existing Geeklog Plugin APIs rather than create a separate integration API for every consumer.

For capabilities that go beyond normalized content (dashboard summaries, diagnostics, navigation, specialized services and actions), use the shared [`plugin-capability-contract.md`](plugin-capability-contract.md). The capability layer advertises existing provider-owned surfaces; it does not replace Item Info, lifecycle events, search, feeds or services.

> This document describes a recommended interoperability contract. Some capabilities already exist in Geeklog, while collection filtering conventions described here are proposed harmonization rules and are not currently universal Geeklog core requirements.

---

## Compatibility target

For plugins currently being modernized, the working target remains:

- **Geeklog 2.1.1 through 2.2.2**
- **PHP 5.6 through PHP 8.1**

The interoperability layer should therefore use the common safe subset of both Geeklog and PHP generations wherever practical.

Geeklog 2.2.2 provides additional lifecycle and URL-resolution capabilities that may not exist in the same form in Geeklog 2.1.1. New plugins should use those capabilities when available while retaining safe fallbacks for older supported versions.

---

# 1. Expose content through `plugin_getiteminfo_*()`

A content plugin should implement:

```php
function plugin_getiteminfo_PLUGIN($id, $what, $uid = 0, $options = array())
```

Example for Maps:

```php
function plugin_getiteminfo_maps($id, $what, $uid = 0, $options = array())
```

This should be the primary structured metadata interface used by other plugins.

At minimum, an addressable content item should be able to expose:

```text
id
title
url
description or excerpt
date-created
date-modified
```

Where meaningful, a plugin may also expose:

```text
uid
author
image
category
topic
type
subtype
hits
```

The exact property set may vary by plugin, but commonly useful names should be kept consistent.

`hits` is the recommended canonical field for a persisted per-item view counter when the plugin maintains one. A plugin may use another internal column name, but consumers should not need to know that implementation detail.

## Single-property compatibility with native Geeklog consumers

**Provider contract:** For a concrete item ID (not `'*'`), when `$what` requests exactly one property, return its **scalar value** (for example a URL string for `'url'`, a title string for `'title'`, or an empty string when unavailable). Do not return a one-element numeric or associative array. This preserves compatibility with native Geeklog 2.1.1/2.2.2 consumers, including the What's New comments renderer, which concatenates `PLG_getItemInfo($type, $id, 'url')` directly with `'#comments'`. Returning an array produces an *Array to string conversion* warning.

For multiple requested properties on a concrete item, preserve the ordered positional result expected by existing Geeklog callers. For `$id === '*'`, retain the established **collection of records** contract, including when the collection requests one property; do not flatten collections into a scalar.

**Consumer compatibility:** Modern consumers (FAQ, Hub, Hello, Documents, Videos, Maps and other integrations) must continue accepting legacy single-property results in scalar, numeric-array and associative-array forms. This tolerance exists for older providers and **does not remove the scalar requirement for newly modernized providers**. Normalize at the consumer boundary rather than weakening native compatibility.

Provider acceptance tests (run through the real `PLG_getItemInfo` dispatcher under appropriate permissions):

```php
$url = PLG_getItemInfo('PLUGIN', $publicId, 'url', $uid);
$title = PLG_getItemInfo('PLUGIN', $publicId, 'title', $uid);
assert(is_string($url) && $url !== '');
assert(is_string($title));
assert(is_string($url . '#comments'));

$fields = PLG_getItemInfo('PLUGIN', $publicId, 'id,title,url', $uid);
assert(is_array($fields) && count($fields) === 3);
assert($fields[2] === $url);

$items = PLG_getItemInfo('PLUGIN', '*', 'id,title,url', $uid);
assert(is_array($items)); // Collection remains structured
```

Additionally, exercise a real public page with new comments in the native **What's New** block, and verify there are no PHP warnings. Repeat single-property checks for roots/categories and for inaccessible or missing items when the plugin supports those identities, verifying no restricted metadata leaks. Keep separate regression tests for legacy consumers accepting array-shaped responses.

## Why this matters

A consumer should be able to request content metadata without accessing plugin tables directly.

For example, Hub or Hello should not need to know whether Maps stores a title in `map_name`, `title`, or another internal column.

The plugin remains responsible for:

- permissions;
- SQL structure;
- URL construction;
- content formatting;
- plugin-specific rules.

The consumer receives only the normalized information it requested.

## Addressable public resources are not limited to leaf items

An Item Info content provider may expose any **stable public addressable resource** that it owns, not only terminal/leaf content.

Examples include:

```text
root / catalogue page
category
album
forum
channel
map
marker
topic
product
classified
contact form landing page
terminal content item
```

A resource is appropriate for the content contract when it has a stable provider-owned identity, a meaningful public URL, and permission-aware visibility.

Consumers must therefore not assume that every Item Info record is a leaf item.

Recommended normalized fields for addressable resources are:

```text
id
title
url
type
subtype
is-container
parent-id
parent-subtype
```

The first three fields remain the practical minimum for discovery. `subtype` is strongly recommended when one provider exposes more than one addressable object family. `is-container`, `parent-id`, and `parent-subtype` are optional additive fields that allow consumers to understand lightweight hierarchy without reading provider tables.

Examples:

```text
documents:root                 subtype=root       is-container=1
documents:category:12          subtype=category   is-container=1
documents:3-airbus-a321-neo    subtype=document   is-container=0

mediagallery:root              subtype=root       is-container=1
mediagallery:album:45          subtype=album      is-container=1
mediagallery:media:987         subtype=media      is-container=0

forum:root                     subtype=root       is-container=1
forum:category:3               subtype=category   is-container=1
forum:forum:8                  subtype=forum      is-container=1
forum:topic:123                subtype=topic      is-container=0
```

### Stable identity when subtype is not transported separately

Some Geeklog APIs, including `PLG_itemDisplay($id, $type)`, do not transport a separate subtype.

When one provider exposes several addressable object families, the provider-owned `id` should therefore remain unambiguous on its own.

Recommended patterns include:

```text
root
category:12
album:45
forum:8
topic:123
channel:UC...
marker:27
```

Consumers must not invent these namespaces. The owning provider defines and documents its stable public identities.

### Addressable means URL-addressable

A resource exposed through Item Info should correspond to a public state that can be reached again from its provider-owned URL.

Do not expose a container or filtered view as a stable content resource when its identity depends only on transient request state such as:

```text
POST-only form state
cookie-only filters
session-only navigation state
temporary UI selections
```

If a category, catalogue view, album, forum or similar container is meant to be addressable by other plugins, give it a stable canonical GET URL and make the same provider-owned identity resolve back to that URL.

For example:

```text
id = category:12
url = /classifieds/index.php?catid=12
```

is interoperable, while a category that exists only because the browser previously submitted a form or retained a cookie is not a stable cross-plugin resource.

This matters especially for stored relationships. A consumer may persist `provider + item_id` for months or years; resolving that identity later must not depend on invisible browser state from the original request.

### Root/catalogue resources

A plugin with a stable public landing page may expose that page as an addressable resource even when it has no database row.

For example, Contact may expose:

```text
provider = contact
id = root
subtype = contact-form
title = Contact
url = /contact/
is-container = 1
```

This does not require Contact to become a database-backed editorial content system. It only means the plugin owns a stable public resource that other interoperable consumers can identify.

---

# 2. Support collection retrieval

For interoperability use cases such as newsletters, recent-content lists, hubs, dashboards, and recommendations, retrieving one item is not enough.

A modernized content plugin should support the existing Geeklog convention of using:

```php
$id = '*';
```

when its `plugin_getiteminfo_*()` implementation is capable of returning multiple items.

Example:

```php
plugin_getiteminfo_maps(
    '*',
    'id,title,url,excerpt,date-modified',
    0,
    $options
);
```

When `'*'` is used, the function should return an array of normalized content records rather than one record.

## Proposed common collection options

The `$options` parameter should gradually be harmonized across modernized content plugins.

Recommended initial options are:

```php
$options = array(
    'since' => $timestamp,
    'limit' => 20,
    'order' => 'modified-desc'
);
```

Suggested meanings:

- `since` — return items created or modified at or after this timestamp;
- `limit` — maximum number of items to return;
- `order` — requested ordering, initially supporting values such as `modified-desc`, `created-desc`, or `hits-desc` when a plugin exposes per-item view counts.

Possible later extensions may include:

```text
until
author
topic
category
subtype
ids
parent-id
parent-subtype
is-container
```

When a provider exposes multiple addressable subtypes, collection filtering by `subtype` is recommended so consumers can request only categories, albums, forums, terminal items, or another provider-owned object family without loading the provider's complete public namespace.

These filtering options are **recommended interoperability conventions**, not a claim that current Geeklog core already enforces them.

## Per-item view counts and popular-content collections

Content plugins that maintain a per-item view counter should expose it through the same Item Info contract rather than require dashboards or other consumers to read plugin tables directly.

The canonical interoperability field is:

```text
hits
```

`hits` represents the plugin's own persisted view or visit count for one addressable content item. A plugin may keep a differently named internal column such as `views`, `sp_hits`, `video_views`, or similar; that internal name should not leak into the interoperability contract.

The field is optional. Plugins that do not track per-item views should simply omit it. Consumers must not treat a missing `hits` value as zero unless that behavior is explicitly appropriate for their use case.

When `hits` is supported, the plugin should expose it for both single-item requests and collection requests where practical:

```php
$item = PLG_getItemInfo(
    'PLUGIN',
    $id,
    'id,title,url,hits',
    0,
    array()
);
```

and:

```php
$items = PLG_getItemInfo(
    'PLUGIN',
    '*',
    'id,title,url,hits,type',
    0,
    array(
        'limit' => 10,
        'order' => 'hits-desc'
    )
);
```

Recommended semantics for `hits-desc` are:

- return only addressable items the requesting user is allowed to see;
- order by the normalized `hits` value from highest to lowest;
- honor `limit`;
- keep URL generation and permission checks inside the plugin;
- avoid exposing draft, private, disabled, or otherwise inaccessible items merely because they have historical hits.

This supports consumers such as administration dashboards, reporting tools, Hub, recommendation features, and future analytics views without coupling them to plugin-specific SQL.

### Aggregate statistics are a different contract

Per-item popularity and plugin-wide statistics should remain distinct:

```text
plugin_statssummary_*() / plugin_showstats_*()
    -> aggregate or presentation-oriented plugin statistics

plugin_getiteminfo_*('*', ..., order = hits-desc)
    -> structured list of the plugin's most-viewed individual content items
```

A plugin may implement either capability or both. Implementing native statistics callbacks does not by itself provide a structured popular-content collection. Conversely, exposing `hits` through Item Info does not replace the native `/stats.php` callbacks.

Geeklog core content already demonstrates this distinction: stories have a per-item `hits` value, while Static Pages maintain `sp_hits` internally. A modernized interoperability layer should normalize such counters to `hits` for consumers.

---

# 3. Emit lifecycle events consistently

Content plugins should notify Geeklog when addressable content changes.

## Save or update

After a successful creation or modification:

```php
PLG_itemSaved($id, 'PLUGIN');
```

Example:

```php
PLG_itemSaved($map_id, 'maps');
```

## Delete

After a successful deletion:

```php
PLG_itemDeleted($id, 'PLUGIN');
```

Example:

```php
PLG_itemDeleted($map_id, 'maps');
```

These calls should be present on **all important mutation paths**, including alternate administration paths, services, imports, moderation paths, or other save/delete routes where applicable.

## Why lifecycle events matter

Hello can periodically request recent content.

Hub has a different need: it should be able to react when content changes rather than repeatedly scan every plugin table.

A typical Hub flow can therefore become:

```text
Content saved
    ↓
PLG_itemSaved()
    ↓
Hub receives the event
    ↓
Hub asks the plugin for structured item information
    ↓
Relations, URLs, indexes or dependent pages can be refreshed
```

This separates **change detection** from **content retrieval**.

Lifecycle events are also useful to consumers such as XMLSitemap, which can react to saves and deletions for content types it tracks.

---

# 4. Preserve `sub_type` when available

Geeklog 2.1.1 and Geeklog 2.2.2 do not expose exactly the same lifecycle callback contract.

Older supported installations may use a legacy callback form without `sub_type`, while Geeklog 2.2.2 can provide a subtype-aware lifecycle contract.

Interoperability consumers such as Hub should therefore model identity as:

```text
plugin
item_id
item_sub_type = NULL when unavailable
```

A plugin should not require `sub_type` for its basic interoperability behavior unless the content model genuinely depends on it.

This allows the same logical relation model to work across Geeklog 2.1.1 and 2.2.2.

---

# 5. Expose item-to-URL resolution when supported

For Geeklog 2.2.2, modernized content plugins should implement:

```php
function plugin_idtourl_PLUGIN($sub_type, $item_id)
```

Example:

```php
function plugin_idtourl_maps($sub_type, $item_id)
{
    // Return the canonical URL for the requested map item.
}
```

This gives consumers a standard way to resolve an item URL without duplicating plugin routing rules.

Potential consumers include:

- Hub;
- IndexNow;
- sitemap generation;
- notifications;
- logs and audit tools.

## Compatibility fallback

Because this capability is not available in the same form across the full Geeklog 2.1.1–2.2.2 range, the item URL should also remain obtainable through:

```php
plugin_getiteminfo_PLUGIN(..., 'url', ...)
```

Consumers should treat `plugin_idtourl_*()` as an additional capability and fall back to Item Info when necessary.

---

# 6. Expose a generic public item extension point with `PLG_itemDisplay()`

For addressable content that has a normal full public view, modernized plugins should expose a stable extension point by calling Geeklog's existing:

```php
PLG_itemDisplay($id, $type)
```

This dispatcher exists in **Geeklog 2.1.1 and Geeklog 2.2.2**, so it can be used across the current transition compatibility range without introducing a Hub-specific API or a separate 2.1.1 fallback.

Geeklog calls every active:

```php
plugin_itemdisplay_PLUGIN($id, $type)
```

implementation and returns the successful display fragments to the content owner. The owning plugin should render those fragments at a stable location on the **full public item page**, normally after the main item content and before secondary UI such as comments, navigation or administrative actions.

Conceptually:

```text
provider-owned item content
        ↓
PLG_itemDisplay($id, $type)
        ↓
third-party contextual fragments
        ↓
comments / secondary actions
```

Example:

```php
$extensions = PLG_itemDisplay($itemId, 'videos');

foreach ($extensions as $extensionHtml) {
    $content .= $extensionHtml;
}
```

The exact integration should fit the provider's template/rendering architecture; the important rule is that the provider owns the placement while third-party plugins contribute through Geeklog's generic dispatcher.

## Why this matters

This enables reusable cross-plugin presentation without coupling the provider to a specific consumer.

For example, Videos should not know that Hub exists. Videos only reports that it is displaying `videos:<id>`. Hub may then implement:

```php
function plugin_itemdisplay_hub($id, $type)
{
    // Return contextual Hub markup when this item belongs to a pillar.
}
```

The same extension point can be reused by other plugins later.

Recommended uses include:

- Hub pillar backlinks;
- contextual relationship/navigation fragments;
- provider-independent annotations or related presentation supplied by another plugin;
- future integrations that need a safe server-rendered placement point.

A full public resource view may be a leaf item or a container. For example, a provider may legitimately expose one `PLG_itemDisplay()` insertion point on:

```text
root / catalogue page
category page
album page
forum page
terminal item page
```

The important distinction is **full resource view versus repeated presentation**. Call the dispatcher once for the page-level resource being viewed; do not call it for every card, row, thumbnail or search result contained inside that page.

## Provider rules

A content plugin implementing this placement should:

- call `PLG_itemDisplay()` only for the normal **full resource view**, not for every list/card/search result;
- pass the stable content identity used by its Item Info contract;
- render returned fragments server-side at a predictable location;
- keep the provider responsible for its own page layout, permissions and primary content;
- when a leaf resource depends on a container that has its own ACL or publication state, keep hierarchical visibility consistent across Item Info collection, single-item lookup, URL resolution and the rendered page; a leaf must not become discoverable through one contract while its owning category/container is hidden through another;
- remain fully functional when no extension fragment is returned;
- avoid Hub-specific callbacks, direct Hub table access, DOM injection or JavaScript-only insertion;
- avoid querying another plugin's private tables to construct the extension content.

This is a **generic Geeklog interoperability point**, not a requirement to depend on Hub.

## Consumer schema safety at the extension boundary

A provider that correctly calls:

```php
PLG_itemDisplay($id, $type)
```

must not become unavailable because a consuming plugin has an older or partially migrated version of **its own** persistence schema.

This is especially important for relation-capable consumers such as FAQ, Hub, annotations, recommendation engines, or other plugins that receive `PLG_itemDisplay()` calls from many unrelated providers. A schema assumption inside the consumer can otherwise make every participating provider fail even though those providers are correctly implementing the Geeklog extension contract.

Therefore a consumer invoked through `PLG_itemDisplay()` should:

- treat its own persisted schema as an internal compatibility boundary;
- avoid assuming that every optional/new relation column already exists merely because the new code files are present;
- when supporting an older persisted state during an upgrade window, inspect the actual schema or use one centralized schema-capability helper before constructing SQL that references newer columns;
- provide documented compatibility defaults for fields that did not exist in the older supported state;
- keep those fallbacks inside the consumer's data-access layer rather than adding provider-specific exceptions;
- return no fragment, or a reduced but valid fragment, when the consumer cannot safely resolve its own contextual data;
- never require the provider to know which consumer schema version is installed.

For example, if a newer consumer relation table adds fields such as `topic_scope` or `sort_order`, code deployed before the database migration completes must not issue unconditional queries against those columns from the public extension callback.

This rule is **not** permission to leave schemas indefinitely half-migrated. The normal upgrade path must still converge on the current schema. The compatibility layer exists to keep shared-file deployments, interrupted upgrades, and supported transitional states from breaking unrelated provider pages.

The same principle applies to internal query expressions: if a compatibility helper has already resolved a safe field expression for the current schema, downstream SQL must consistently use that resolved expression rather than later referencing the new column directly.

## Identity and subtype caution

The current dispatcher signature is:

```php
PLG_itemDisplay($id, $type)
```

It does not carry a separate `sub_type`.

Providers exposing several independently addressable object families must therefore ensure that the `type + id` pair passed to the dispatcher identifies the displayed object unambiguously, or document a provider-neutral identity convention before relying on this hook for subtype-specific relationships.

This matters especially for plugins such as Maps where maps and markers may both become first-class content objects. Hub and other consumers should not invent provider-private identifiers merely to work around an ambiguous public identity.

---


## Contextual relationship placement and manual rendering

Cross-plugin relationships may exist independently from automatic public injection.

A relation-capable consumer should therefore distinguish between:

- **automatic placement** — the related fragment is rendered through the provider's native extension point;
- **manual-only placement** — the relation remains stored and permission-aware, but no automatic fragment is injected;
- **explicit contextual rendering** — an author deliberately inserts the related fragment inside the owning content.

Manual-only must not mean "relation disabled" or "data ignored". It means that presentation is author-controlled.

When the content format supports Geeklog autotags, a consumer may expose a contextual autotag that renders the relationships of the current content without requiring the author to repeat its identity. Conceptually:

```text
[consumer-context]
```

The autotag should use the content context already supplied by Geeklog whenever possible. In particular, when `PLG_replaceTags()` is called with a provider/type and content id, that `provider + id` pair should be treated as the authoritative current-content identity.

Do not reconstruct the identity from URL patterns when Geeklog has already supplied it.

Compatibility fallbacks based on the request route are acceptable only when:

- the supported Geeklog call path does not provide the context;
- the fallback is limited to a known provider-owned route;
- the provider/id mapping is unambiguous;
- the fallback is isolated and documented;
- failure to resolve the context does not leak technical error text to public visitors.

Explicit autotags that name a provider and id may also remain available for advanced or legacy cases, for example:

```text
[consumer-related:article my-story-id]
```

The contextual and explicit forms must use the same relationship engine as automatic rendering. They must therefore preserve the same:

- ACL checks;
- enabled/disabled state;
- ordering;
- duplicate suppression;
- inheritance rules;
- conflict policy;
- provider-owned identity rules.

A manual placement path must not become a shortcut around the business rules used by automatic placement.

Provider-specific placement choices may expose only the positions that the provider can reliably support. For example, a provider with one native item-display insertion point may offer only `automatic` and `manual`, while another provider may expose several stable page-level positions. Consumers should model these as provider capabilities rather than hard-code one global set of positions for every content type.

# 7. Keep What's New as a presentation capability

Plugins whose content belongs in Geeklog's native **What's New** block may also implement:

```php
plugin_whatsnewsupported_PLUGIN()
plugin_getwhatsnew_PLUGIN()
```

Example for Maps:

```php
plugin_whatsnewsupported_maps()
plugin_getwhatsnew_maps()
```

These APIs remain useful and should be preserved where appropriate.

However, `plugin_getwhatsnew_*()` typically returns presentation-ready HTML. It should therefore not become the main data contract for Hello, Hub, or other structured consumers.

The preferred model is:

```text
plugin_getiteminfo_*()
        ↓
structured content metadata
        ↓
 ┌──────┼───────────────┐
 ↓      ↓               ↓
Hello   Hub        What's New renderer
```

What's New can reuse the same underlying plugin query logic while remaining responsible for HTML output.

---

# 8. Optional and distribution capabilities

Once the core interoperability layer is stable, plugins may add additional capabilities according to their role.

These capabilities should be detected and documented independently from the basic Hub/content readiness score. A plugin does not need every capability to be interoperable.

## Related items

```php
plugin_getrelateditems_PLUGIN()
```

Useful for:

- Hub suggestions;
- related-content modules;
- recommendation systems;
- navigation between semantically connected resources.

## Search

```php
plugin_dopluginsearch_PLUGIN()
```

Useful when plugin content should participate in Geeklog search.

## Autotags

```php
plugin_autotags_PLUGIN()
```

Useful for embedding plugin content in stories, static pages, or other supported content.

## Blocks

```php
plugin_getBlocks_PLUGIN()
```

Useful where the plugin provides reusable dynamic blocks.

## Statistics

Plugins may contribute to Geeklog's native statistics system and `/stats.php` through:

```php
plugin_showstats_PLUGIN()
plugin_statssummary_PLUGIN()
```

These callbacks represent a native aggregation contract: the plugin remains responsible for calculating and formatting its statistics while Geeklog provides the common statistics surface.

A useful capability classification is:

```text
Full     = both callbacks are implemented
Partial  = one callback is implemented
None     = neither callback is implemented
```

Statistics support is **not required for basic content interoperability** and should not be part of the core Hub readiness score. It is nevertheless valuable for dashboards, administration, reporting, and future Hub aggregation features.

Consumers should prefer these native callbacks over direct reads of another plugin's tables when the goal is to obtain the plugin's own statistical presentation or summary.

For a structured list of individual popular items, consumers should instead request the optional `hits` field through `plugin_getiteminfo_*()` collection support and use `order => 'hits-desc'` when the plugin supports it. This avoids overloading presentation-oriented statistics callbacks with a second data-contract role.

## Content Syndication

Plugins that should be available through Geeklog's **Content Syndication** administration (`/admin/syndication.php`) can expose native feed support through:

```php
plugin_getfeednames_PLUGIN()
plugin_getfeedcontent_PLUGIN()
```

`plugin_getfeednames_PLUGIN()` exposes the feed choices a plugin makes available, while `plugin_getfeedcontent_PLUGIN()` supplies the feed entries and metadata used by Geeklog to generate RSS/Atom output.

A plugin may additionally implement:

```php
plugin_feedupdatecheck_PLUGIN()
```

This callback lets Geeklog determine efficiently whether an existing feed needs to be regenerated after content changes.

Recommended capability classification:

```text
Full     = getFeedNames + getFeedContent
Partial  = only one of the two primary callbacks
None     = neither primary callback
```

`plugin_feedupdatecheck_*()` is an optional optimization and should be reported separately rather than required for Full syndication support.

Content Syndication is a distribution capability, not a replacement for `plugin_getiteminfo_*()`. Feed rendering and structured inter-plugin metadata serve different consumers and should remain separate.

## XML Sitemap contribution

Geeklog's XMLSitemap integration has a specific collection contract that should be recognized explicitly.

Since Geeklog 2.1.1, sitemap collection can use:

```php
plugin_collectSitemapItems_PLUGIN($uid, $limit)
```

through the core dispatcher:

```php
PLG_collectSitemapItems($type, $uid, $limit)
```

A plugin implementing `plugin_collectSitemapItems_*()` provides the **native sitemap collector** for its content type.

If the plugin does not provide that callback, the XMLSitemap plugin can fall back to collection through:

```php
PLG_getItemInfo($type, '*', $what, $uid, $options)
```

Therefore sitemap interoperability should distinguish at least:

```text
Native collector      = plugin_collectSitemapItems_PLUGIN() exists
Item Info fallback    = collection can be obtained through plugin_getiteminfo_PLUGIN('*', ...)
None detected         = neither path is available
```

This distinction matters because implementing `plugin_getiteminfo_*()` with collection support already makes a content plugin useful to XMLSitemap even when it does not implement the specialized sitemap collector.

The specialized collector remains preferable when sitemap-specific behavior, permissions, filtering, priorities, or performance considerations justify it.

### Native sitemap collector requirements

When a plugin implements `plugin_collectSitemapItems_PLUGIN($uid, $limit)`, the callback should be treated as the canonical sitemap-oriented view of the plugin's public content.

The collector should:

- return only public, indexable, provider-owned canonical GET URLs;
- never return wildcard, placeholder, session-dependent, POST-only, or generic collection URLs such as `id=*` or `s=*`;
- deduplicate URLs before returning them, including content that can belong to multiple containers or relationships;
- apply anonymous/public permission rules consistently with the plugin's normal frontend access model;
- exclude hidden, disabled, unpublished, unreleased, expired, or otherwise non-indexable resources;
- distinguish addressable containers from leaf items when both are intended to be indexed, for example plugin root, category/album, and individual item;
- provide `date-modified`, `change-freq`, and `priority` when the plugin has meaningful sitemap-specific values for them;
- honor the `$limit` parameter without returning duplicate entries;
- avoid relying on UI state or generic collection behavior when a dedicated sitemap query can provide a safer and more deterministic result;
- remain safe during plugin lifecycle transitions. In particular, XMLSitemap may be invoked from a plugin state-change callback before Geeklog has refreshed `$_PLUGINS`. A collector must not assume that the plugin is already fully reflected in the enabled-plugin runtime state during that transient window.

For content that can appear in more than one container, one canonical resource URL should normally be emitted once. Container pages may be emitted separately only when they are themselves stable, public, indexable resources.

Example hierarchy:

```text
plugin root
  -> category / album / container
     -> individual content item
```

The native collector should not merely expose every row returned by an internal relationship join. It should expose the canonical sitemap representation of the provider's public content.

Lifecycle notifications complement this collection contract: XMLSitemap may listen to `PLG_itemSaved()` and `PLG_itemDeleted()` to know when tracked content changes, while the collector or Item Info interface remains responsible for supplying the addressable content itself.

## Services

Geeklog services should be added when another plugin genuinely needs a specialized action, transaction, or rendering capability that is not covered by content metadata.

Services should not be introduced merely to duplicate `plugin_getiteminfo_*()`.

### Exact source-field inspection and controlled mutation

Some consumers need the exact stored/editorial source rather than a normalized Item Info representation. Examples include historical-autotag audits, content migration tools, language audits and controlled refactoring.

That need must remain distinct from ordinary `content.read`.

Plugins that support such workflows should follow the shared source-field contract in:

[`plugin-source-field-audit-mutation-contract.md`](plugin-source-field-audit-mutation-contract.md)

Key rules:

- expose exact source fields only through an explicit capability;
- keep read and write capabilities separate;
- expose stable provider-owned field identifiers instead of SQL column names;
- keep permissions, validation, save logic, lifecycle notifications and cache invalidation inside the owning plugin;
- do not require consumers such as AdSense to read or update private plugin tables;
- support bounded collection/audit access for large installations where practical.

The initial reference consumer is AdSense, which needs to find and optionally remove historical tags such as `[adsense:1]` and `[leaderboard:1]` without rewriting unrelated content.

---

# 9. Recommended implementation priorities

| Priority | Capability | Purpose |
| --- | --- | --- |
| **P1** | `plugin_getiteminfo_PLUGIN()` | Expose structured content metadata |
| **P1** | `'*'` collection support | Expose multiple content items and provide the XMLSitemap fallback path |
| **P1** | `since`, `limit`, `order` options | Retrieve recent, changed, or ordered content |
| **P1** | `PLG_itemSaved()` | Signal create/update lifecycle changes |
| **P1** | `PLG_itemDeleted()` | Signal deletions |
| **P2** | optional `hits` field + `hits-desc` ordering | Expose per-item popularity to dashboards and other structured consumers when the plugin tracks views |
| **P2** | public `PLG_itemDisplay($id, $type)` placement | Allow generic server-rendered contextual fragments on full item views |
| **P2** | `plugin_idtourl_PLUGIN()` | Resolve canonical item URLs where supported |
| **P2** | `plugin_collectSitemapItems_PLUGIN()` | Provide optimized/native XML Sitemap collection where useful |
| **P2/P3** | `plugin_getfeednames_PLUGIN()` + `plugin_getfeedcontent_PLUGIN()` | Participate in Content Syndication when the content type is feed-worthy |
| **P3** | `plugin_feedupdatecheck_PLUGIN()` | Optimize feed regeneration |
| **P3** | `plugin_showstats_PLUGIN()` + `plugin_statssummary_PLUGIN()` | Contribute to native Geeklog statistics where meaningful |
| **P3** | `plugin_whatsnewsupported_PLUGIN()` | Participate in Geeklog What's New |
| **P3** | `plugin_getwhatsnew_PLUGIN()` | Render recent items in What's New |
| Future | `plugin_getrelateditems_PLUGIN()` | Support Hub relations and recommendations |

For the next modernization work on **Maps, Documents, Videos, Store**, and similar content plugins, the P1 capabilities should be treated as the common interoperability baseline. Plugins that already maintain per-item view counters should additionally implement the P2 `hits` capability so dashboards and other consumers can rank content without direct SQL access. Other P2/P3 distribution capabilities should be added when they fit the plugin's role and user value rather than mechanically implemented everywhere.

---

# 10. Example target for Maps

A Maps modernization should aim to expose at least:

```php
function plugin_getiteminfo_maps($id, $what, $uid = 0, $options = array())
```

with support for:

```text
id
title
url
description or excerpt
date-created
date-modified
hits (when Maps maintains a per-item view counter)
```

and collection queries such as:

```php
$options = array(
    'since' => $timestamp,
    'limit' => 20,
    'order' => 'modified-desc'
);

$items = plugin_getiteminfo_maps(
    '*',
    'id,title,url,excerpt,date-modified',
    0,
    $options
);
```

If Maps tracks views per map, it should additionally support the normalized popularity query:

```php
$popular = plugin_getiteminfo_maps(
    '*',
    'id,title,url,hits,type',
    0,
    array('limit' => 10, 'order' => 'hits-desc')
);
```

Maps should also emit:

```php
PLG_itemSaved($map_id, 'maps');
PLG_itemDeleted($map_id, 'maps');
```

and, where supported:

```php
function plugin_idtourl_maps($sub_type, $item_id)
```

If Maps should participate directly in XMLSitemap with sitemap-specific collection logic, it may also implement:

```php
function plugin_collectSitemapItems_maps($uid, $limit)
```

Otherwise, a complete `plugin_getiteminfo_maps('*', ...)` implementation provides the supported XMLSitemap fallback path.

If Maps content is useful as an RSS/Atom source, it may additionally expose:

```php
plugin_getfeednames_maps()
plugin_getfeedcontent_maps()
plugin_feedupdatecheck_maps() // optional optimization
```

If Maps has meaningful site-level statistics, it may contribute them through:

```php
plugin_showstats_maps()
plugin_statssummary_maps()
```

The same pattern can then be applied to Documents, Videos, Store, and other addressable content plugins, while only implementing optional distribution/presentation capabilities that make sense for each plugin.

---

# 11. Consumer responsibilities

The contract also places requirements on consumers.

## Reference consumer profile: FAQ contextual associations

When modernizing a content plugin that should be selectable by the FAQ plugin, do not stop after implementing a callback that merely resembles the Memorandum contract. Validate the provider against the **actual consumer path** used by FAQ.

FAQ currently consumes provider content in four distinct ways.

### 1. Provider discovery

FAQ discovers a third-party provider only when the plugin is active and exposes:

```php
plugin_getiteminfo_PLUGIN()
```

Therefore a capability declaration alone is not enough for FAQ association discovery.

### 2. Selectable content collection

For the administration association picker, FAQ requests:

```php
PLG_getItemInfo(
    'PLUGIN',
    '*',
    'id,title,url,subtype,type',
    0,
    array(
        'limit' => 100,
        'order' => 'modified-desc'
    )
);
```

A provider intended to work smoothly with FAQ should therefore verify this exact collection call.

FAQ accepts collection records in either of these forms:

```php
array(
    'id'      => 'category:12',
    'title'   => 'Guides',
    'url'     => 'https://example.test/documents/category/12',
    'subtype' => 'category'
)
```

or the historical positional shape:

```php
array(
    'category:12',
    'Guides',
    'https://example.test/documents/category/12'
)
```

For associative records, FAQ reads `subtype` first and falls back to `type` when `subtype` is empty.

The practical compatibility target is therefore:

- stable non-empty `id`;
- useful `title`;
- public/canonical `url` when available;
- `subtype` for multi-object providers;
- permission-aware collection results;
- safe support for `limit`;
- graceful handling of the requested `order` even when the provider cannot implement every ordering mode exactly.

### 3. Concrete item title and URL resolution

When displaying stored associations in administration, FAQ resolves the target through the provider rather than reading provider tables.

The resolution order is:

```text
URL
    1. plugin_idtourl_PLUGIN($subtype, $item_id)
    2. PLG_getItemInfo(PLUGIN, item_id, 'url', current_user_uid)

Title
    1. PLG_getItemInfo(PLUGIN, item_id, 'title', current_user_uid)
```

For single-field Item Info requests, FAQ deliberately accepts historical provider return shapes:

```text
scalar
associative array keyed by field
numeric array whose first value is the requested field
```

A provider should still prefer one consistent documented Item Info behavior, but it must be tested through Geeklog's dispatcher rather than only by directly calling the plugin callback.

A provider that works only for a multi-field direct callback test but fails for:

```php
PLG_getItemInfo('PLUGIN', $id, 'title', $uid);
PLG_getItemInfo('PLUGIN', $id, 'url', $uid);
```

is not fully aligned with the current FAQ consumer.

### 4. Public contextual rendering

For automatic contextual FAQ rendering, the content owner must expose the generic Geeklog extension point on the relevant public surface:

```php
PLG_itemDisplay($stableId, 'PLUGIN')
```

The provider owns where the returned fragments are inserted.

FAQ does not need to know the provider's template, route, table or page structure. It matches the provider/type and stable item ID stored in the association.

If the provider exposes several public object families, the ID passed to `PLG_itemDisplay()` must remain unambiguous because this dispatcher does not carry a separate subtype.

Examples:

```text
root
category:12
album:45
forum:8
topic:123
marker:27
```

The same stable ID should be used consistently by:

- Item Info collection;
- concrete Item Info resolution;
- `plugin_idtourl_PLUGIN()`;
- `PLG_itemDisplay()`;
- lifecycle notifications where applicable.

### FAQ alignment acceptance test

Before declaring a plugin "FAQ interoperable", test the following through a real Geeklog runtime:

```text
[ ] plugin appears in FAQ Associations provider list
[ ] FAQ can enumerate its selectable objects without provider-specific SQL
[ ] each returned object has the expected stable ID
[ ] title is readable through a single-field Item Info request
[ ] URL resolves through plugin_idtourl_*() or single-field Item Info
[ ] subtype is exposed for multi-object providers
[ ] current-user administration lookup respects permissions
[ ] an association can be saved and redisplayed with a human-readable linked title
[ ] the provider calls PLG_itemDisplay() on the intended public page
[ ] the consumer still handles the previous supported/partially migrated consumer schema without breaking the provider page
[ ] the saved FAQ/category renders on that page
[ ] the same identity works on Geeklog 2.1.1 without requiring subtype transport
[ ] Geeklog 2.2.2 may additionally use subtype-aware callbacks without changing the stable ID
```

This is a **consumer acceptance profile**, not a FAQ-specific API. The provider still implements generic Geeklog/Memorandum contracts. FAQ is simply a concrete reference consumer used to prove that those contracts work end-to-end.

If another consumer such as Hub or Agent exercises a different part of the shared contract, test that consumer's actual call path as well. "The callback exists" is not sufficient interoperability evidence.


## Hello

Hello should:

- request structured content rather than read another plugin's SQL tables;
- use collection filters to retrieve items since the last digest;
- use Item Info fields to build email content;
- ignore unsupported plugins gracefully;
- preserve each plugin's permission model.

## Hub

Hub should:

- treat `plugin + item_id + nullable item_sub_type` as the external content identity;
- listen to lifecycle events rather than poll plugin tables where practical;
- retrieve metadata through Item Info after an event;
- resolve URLs through `plugin_idtourl_*()` when available, with an Item Info fallback;
- audit native Statistics, Content Syndication, XML Sitemap, and per-item `hits` capabilities separately from basic content readiness;
- avoid becoming the owner of another plugin's content.

## Content Syndication

Geeklog's syndication layer should remain the owner of feed generation. Plugins should expose feed choices/content through the native callbacks rather than external consumers duplicating plugin-specific feed SQL.

## XMLSitemap

XMLSitemap should prefer `PLG_collectSitemapItems()` / `plugin_collectSitemapItems_*()` where available and use the supported `PLG_getItemInfo($type, '*', ...)` fallback otherwise.

A plugin should not need XMLSitemap-specific SQL adapters merely to become discoverable if it already exposes a complete collection-capable Item Info contract.

## Other consumers

IndexNow, Search, AI/agent layers, dashboards, and future integrations should reuse the same normalized metadata and native Geeklog callback surfaces wherever practical rather than invent plugin-specific adapters.

Dashboards that rank individual content by popularity should use the optional `hits` field and `hits-desc` collection ordering instead of reading plugin tables directly. If a plugin does not expose that capability, consumers should degrade gracefully rather than infer plugin-specific table or column names.

---

# Design principle

The plugin remains the authority for its own content.

Other plugins should not need to know:

- database tables;
- column names;
- internal file paths;
- routing implementation;
- permission internals;
- business rules.

They should interact through a small, stable Geeklog interoperability surface:

```text
                         Content plugin
                              │
        ┌─────────────────────┼──────────────────────┐
        │                     │                      │
 getItemInfo              lifecycle            distribution
        │              saved / deleted                │
        │                     │            ┌──────────┼──────────┐
        │                     │            │          │          │
        ↓                     ↓          feeds     sitemap     stats
 structured data             Hub       syndication  collector  callbacks
        │
   ┌────┼───────────────────────┐
   ↓    ↓                       ↓
 Hello  Hub          dashboards / IndexNow / other consumers
```

The long-term objective is not to create a Hub-specific, Hello-specific, or Eclipse-specific API.

It is to make Geeklog plugins **interoperable by design**, so multiple consumers can reuse the same content contract and Geeklog's existing distribution surfaces without coupling themselves to plugin internals.
