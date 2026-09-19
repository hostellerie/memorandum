# Geeklog Plugin Capability Contract

Status: **Proposed shared interoperability contract**

## Purpose

This document defines a shared, provider-neutral capability contract for Geeklog plugins, Core features and themes that need to advertise what they can expose or do without creating consumer-specific APIs.

It complements:

- `plugin-content-interoperability-contract.md` for normalized content;
- `plugin-source-field-audit-mutation-contract.md` for exact source-field inspection and controlled mutation;
- `llm-agent-content-representation-contract.md` for machine-readable resources and agent use;
- `agent-hub-connector-architecture.md` for Agent, Hub and connector responsibility boundaries;
- native Geeklog Plugin APIs and `PLG_invokeService()`.

The main rule is:

> **Declare capabilities once, keep data and business logic in the owning provider, and let Agent, Hub, Eclipse, AdSense, Hello, Monitor and future consumers reuse the same contract.**

This contract must not become a second Plugin API. It describes capabilities and maps them to existing Geeklog APIs, versioned plugin contracts or bounded services.

---

## 1. Why a shared capability contract is needed

Geeklog now has several consumers that need structured information from plugins:

- **Agent** needs provider-neutral resources, capabilities and later authorized actions;
- **Hub** needs content identity, lifecycle, relations, services and diagnostics;
- **Eclipse** needs structured dashboard summaries without querying plugin tables;
- **AdSense** needs content/context and source-field capabilities without depending on Agent;
- **Hello**, **IndexNow**, Sitemap and future consumers need reusable content/lifecycle contracts;
- administration tools need to discover what plugins expose without hard-coded knowledge.

Without one shared contract, each consumer risks creating a parallel registry.

Do not introduce separate registries such as:

```text
Agent capability registry
Hub capability registry
Eclipse dashboard registry
AdSense provider registry
Connector capability registry
```

Prefer one provider-owned declaration consumed by all of them.

---

## 2. Capability declaration

A modernized plugin may expose an optional capability declaration:

```php
function plugin_getcapabilities_PLUGIN()
{
    return array(
        'schema' => 1,
        'roles' => array('content', 'service'),
        'capabilities' => array(
            'content.read',
            'content.collection',
            'content.search',
            'dashboard.summary'
        )
    );
}
```

The exact Geeklog Core callback is not currently universal; this is a memorandum interoperability convention and must be feature-detected.

A capability declaration:

- advertises what the provider supports;
- does not return the actual content or dashboard data;
- does not bypass permissions;
- does not imply write authorization;
- does not require Agent, Hub, Eclipse or another consumer to be installed;
- should remain small and deterministic;
- should use a versioned schema.

Consumers must degrade gracefully when this callback is absent and may infer older Geeklog capabilities where safe.

---

## 3. Roles

Recommended provider roles include:

```text
content
service
presentation
infrastructure
orchestrator
navigation
relationship
diagnostic
```

A plugin may expose more than one role.

Examples:

```text
Videos        -> content, service
Documents     -> content, service
Maps          -> content, service
MediaGallery  -> content, service
Forms         -> service
Monitor       -> diagnostic, service
AdSense       -> service, infrastructure
Hub           -> relationship, orchestrator, service
Menu          -> navigation
Eclipse       -> presentation, consumer
Contact       -> service
```

Roles are descriptive. They must not be converted into mandatory API checklists that penalize a service or presentation component for not exposing content.

---

## 4. Resources, capabilities and actions are different

These concepts must remain separate:

```text
Resources
    readable objects or collections

Capabilities
    operations/features the provider supports

Actions
    state-changing operations the current caller is authorized to execute
```

Examples:

```text
content.read            capability
video:abc123            resource

forms.schema.read       capability
contact-form            resource/definition

forms.submit            action-capable operation
maps.marker.update      action-capable operation
```

A declared capability never means the caller is authorized to execute a write action.

---

## 5. Mapping capabilities to existing Geeklog contracts

The capability layer should reuse existing APIs.

### Content capabilities

```text
content.read
    -> plugin_getiteminfo_PLUGIN()

content.collection
    -> plugin_getiteminfo_PLUGIN('*', ...)

content.search
    -> plugin_dopluginsearch_PLUGIN() or another documented Geeklog search surface

content.popular
    -> Item Info collection with hits + order=hits-desc

content.url.resolve
    -> plugin_idtourl_PLUGIN() and/or Item Info url

content.lifecycle
    -> PLG_itemSaved() / PLG_itemDeleted()

content.related
    -> plugin_getrelateditems_PLUGIN() or Hub relationship services where appropriate

content.syndication
    -> plugin_getfeednames_PLUGIN() / plugin_getfeedcontent_PLUGIN()
```

### Specialized capabilities

Specialized operations should normally use bounded services:

```php
PLG_invokeService($plugin, $service, $args, $output, $svc_msg)
```

Examples:

```text
maps.geo.nearby
maps.marker.list
media.album.list
forms.schema.read
monitor.health
hub.context.read
adsense.ads_txt.status
dashboard.summary
```

A versioned documented plugin function may be used when a service or Item Info is not a natural fit, as with a structured Menu tree.

---

## 6. Canonical capability names

Use stable dotted identifiers.

Initial shared names:

```text
content.read
content.collection
content.search
content.popular
content.related
content.url.resolve
content.lifecycle
content.syndication

content.fields.read
content.source_fields.read
content.source_fields.collection
content.source_fields.update

navigation.read
navigation.tree

dashboard.summary

media.album.list
media.album.read
media.item.read
media.item.collection

maps.map.read
maps.marker.read
maps.marker.list
maps.marker.render
maps.geo.nearby
maps.marker.create
maps.marker.update
maps.marker.validity.set
maps.marker.validity.extend

forms.list
forms.schema.read
forms.submit
forms.submissions.read

monitor.health
monitor.diagnostics
monitor.security.summary
monitor.logs.summary
monitor.plugins.status

adsense.status
adsense.ads_txt.status
adsense.placements.list

contact.form.describe
contact.submit

hub.context.read
hub.related.read
hub.pillar.read
hub.affected.read
hub.integrity.summary
hub.suggestions.read
hub.interoperability.summary
```

Names should be additive and provider-neutral. Do not create equivalent synonyms merely for one consumer.

---

## 7. Dashboard summary contract

`dashboard.summary` is the recommended shared contract for administration dashboards such as Eclipse.

A theme or dashboard must not query plugin tables directly to calculate plugin-specific operational state.

A provider exposing `dashboard.summary` should provide a bounded read-only structured response through a service such as:

```php
PLG_invokeService(
    'videos',
    'dashboard_summary',
    array(),
    $output,
    $svc_msg
);
```

Recommended response shape:

```php
array(
    'schema' => 1,
    'status' => 'ok',
    'metrics' => array(
        array(
            'id' => 'items',
            'label' => 'Items',
            'value' => 128
        )
    ),
    'alerts' => array(
        array(
            'id' => 'storage',
            'status' => 'warning',
            'message' => 'Persistent storage needs attention.'
        )
    ),
    'links' => array(
        array(
            'label' => 'Manage',
            'url' => '...'
        )
    ),
    'updated' => time()
);
```

For current Eclipse 1.2 consumers:

- each metric should provide stable `id`, human-readable `label` and scalar `value`;
- an alert may use either `label` or `message`; `message` is recommended for operational diagnostics;
- an alert may provide `count`; when omitted, Eclipse treats the alert as one actionable condition;
- alert `status` may be `info`, `warning` or `critical`;
- links should point to same-site administration or management surfaces owned by the provider;
- the first valid provider link is used as the default management target for metrics and alerts that do not provide their own URL;
- providers should keep the payload bounded and must not perform external network work merely to render the dashboard.

Recommended top-level fields:

```text
schema
status
metrics
alerts
links
updated
```

Recommended status values:

```text
ok
info
warning
critical
unknown
```

Consumers must treat provider labels and values as presentation data and must not reconstruct the provider's internal logic.

### Dashboard summary versus native statistics

Keep these contracts distinct:

```text
plugin_statssummary_* / plugin_showstats_*
    -> native Geeklog statistics presentation/aggregate statistics

PLG_getItemInfo(... hits ...)
    -> per-item popularity and popular collections

dashboard.summary
    -> operational/admin summary, health, counts, alerts and links
```

A provider may support all three.

---

## 8. Eclipse as a capability consumer

Eclipse is a presentation layer and should be a generic consumer of provider capabilities.

Its administration dashboard should progressively follow:

```text
active plugins
    -> detect capabilities
    -> dashboard.summary available?
    -> PLG_invokeService()
    -> normalize
    -> render Eclipse cards
```

Eclipse must not:

- query Videos, Documents, Maps, MediaGallery, Forms, Monitor, AdSense or Hub private tables;
- duplicate plugin permission logic;
- implement provider-specific health algorithms;
- require plugins to depend on Eclipse.

It may keep compatibility adapters for old plugins, but modern integrations should be capability-driven.

Useful dashboard summaries include:

- Videos: retained videos, channels, moderation state, provider status;
- Documents: documents, categories, drafts, pending submissions;
- Maps: maps, markers, expiring markers, provider status;
- MediaGallery: albums, media, moderation, storage status;
- Forms: active forms and pending/new submissions where authorized;
- Monitor: health, warnings, logs and security summary;
- AdSense: serving mode, placements, ads.txt status and conflicts;
- Hub: pillars, relations, broken relations, suggestions and interoperability status.

Eclipse should aggregate and present. The owning provider calculates the meaning of each value.

### Eclipse 1.2 implemented consumer behavior

The current Eclipse 1.2 dashboard implements this contract generically for active plugins:

```text
active plugin
    -> plugin_getcapabilities_PLUGIN()
    -> dashboard.summary declared?
    -> PLG_invokeService(PLUGIN, 'dashboard_summary', ...)
    -> validate schema/status
    -> normalize metrics / alerts / links
    -> render dashboard
```

Current behavior intentionally avoids provider-specific adapters:

- structured `dashboard.summary` metrics are preferred over legacy `PLG_getPluginStats()` rows for the same provider, preventing duplicate statistics;
- legacy plugin statistics remain as a compatibility fallback when no structured dashboard metrics are exposed;
- metrics are displayed in the generic plugin-content statistics area;
- provider management links are attached to metric labels when available;
- explicit alerts are promoted to the dashboard's **Needs attention** area;
- conventional metric IDs `pending` and `drafts` are also promoted to **Needs attention** when their value is greater than zero;
- `pending` is treated as an operational warning and `drafts` as informational/editorial work;
- one provider failing, denying access or returning an unsupported payload must not break the complete dashboard;
- provider permissions remain authoritative: Eclipse never bypasses them and never queries provider-private tables.

This means a newly modernized plugin can become visible in Eclipse without an Eclipse-specific integration simply by declaring `dashboard.summary` and implementing the bounded service contract.

---

## 9. Hub as consumer and provider

Hub consumes content/provider capabilities but must also expose the context it owns.

Hub should eventually expose:

```text
hub.context.read
hub.related.read
hub.pillar.read
hub.affected.read
hub.integrity.summary
hub.suggestions.read
hub.interoperability.summary
dashboard.summary
```

Hub remains authoritative for:

- pillar relationships;
- cross-plugin relations;
- dependency/affected-item graphs;
- orphan and integrity diagnostics;
- editorial relationship suggestions.

Agent may consume these services when machine requests need contextual information.

Eclipse may consume `dashboard.summary` and `hub.interoperability.summary` for administration presentation.

Hub must not become the content owner and must not recreate plugin business logic.

The Hub interoperability audit should recognize this shared capability contract rather than maintain a separate Hub-only registry.

---

## 10. Agent consumption model

Agent should consume the same shared provider contracts.

Typical mapping:

```text
content.read
    -> normalized Agent resource

content.collection
    -> normalized Agent collection

navigation.tree
    -> navigation provider

hub.context.read
    -> contextual enrichment

maps.geo.nearby
    -> specialized read capability

monitor.health
    -> administrative/diagnostic capability when authorized
```

Agent must not be required for plugin-to-plugin interoperability.

AdSense, Eclipse, Hub or another plugin may consume the same owning-plugin contract independently.

---

## 11. Recommended capability targets by current project

### Videos

Target:

```text
content.read
content.collection
content.search
content.url.resolve
content.lifecycle
content.syndication
content.popular             when meaningful
dashboard.summary
videos.channels.read        optional specialization
videos.rankings.read        optional specialization
videos.provider.status      optional specialization
```

Videos already has a strong Item Info/collection/lifecycle foundation. Capability declaration and dashboard summary should not block release if the stable content contract is already validated, but they are high-value additions.

### Documents

Target:

```text
content.read
content.collection
content.search
content.popular
content.url.resolve
content.lifecycle
content.syndication
content.fields.read
content.source_fields.read
dashboard.summary
```

Later, separately permissioned:

```text
content.source_fields.update
```

Documents should expose structured field information without forcing consumers to understand `documents_values`.

### Maps

Target:

```text
content.read
content.collection
content.search
content.url.resolve
content.lifecycle
dashboard.summary

maps.map.read
maps.marker.read
maps.marker.list
maps.marker.render
maps.geo.nearby
```

Write/action capabilities must remain separate:

```text
maps.marker.create
maps.marker.update
maps.marker.validity.set
maps.marker.validity.extend
```

### MediaGallery

Target:

```text
content.read
content.collection
content.url.resolve
content.lifecycle
dashboard.summary

media.album.list
media.album.read
media.item.read
media.item.collection
```

Use stable subtypes such as `album` and `media`. Media resources should expose useful normalized metadata such as title, URL, description, image/media URL, media type, MIME type where appropriate, album, dates and author/owner subject to permissions.

### Forms

Forms should primarily be a service provider rather than a normal editorial content provider.

Target before a stable 1.0 contract where practical:

```text
forms.list
forms.schema.read
dashboard.summary
```

Later and separately authorized:

```text
forms.submit
forms.submissions.read
```

A form schema should expose stable field IDs, labels, types, required state and options without requiring HTML parsing.

### Monitor

Monitor should not invent Item Info merely for interoperability.

Target:

```text
monitor.health
monitor.diagnostics
dashboard.summary
```

Then:

```text
monitor.security.summary
monitor.logs.summary
monitor.plugins.status
```

Read-only diagnostics should be the first integration surface. Repair/mutation actions remain separate and explicit.

### AdSense

AdSense is primarily a consumer of content/context capabilities, not a content-owning provider.

It may expose:

```text
dashboard.summary
adsense.status
adsense.ads_txt.status
adsense.placements.list
```

AdSense must consume provider contracts directly and must not depend on Agent to reach another plugin.

### Contact

Contact does not need an editorial Item Info provider.

Possible capabilities:

```text
dashboard.summary
contact.form.describe
```

Later:

```text
contact.submit
```

Submission must preserve Contact anti-spam, validation, rate-limit and consent rules.

### Eclipse

Eclipse is primarily a capability consumer and presentation layer.

It should not expose content merely to appear interoperable.

Theme identity/version information may be exposed to diagnostics if useful, but the priority is consuming shared capabilities correctly.

### Hub

Hub is both consumer and relationship/context provider.

Target:

```text
hub.context.read
hub.related.read
hub.pillar.read
hub.affected.read
hub.integrity.summary
hub.suggestions.read
hub.interoperability.summary
dashboard.summary
```

---

## 12. Source-field capabilities

Raw/source-field inspection is not the same as normalized content reading.

Use the dedicated source-field audit/mutation contract.

Capability distinction:

```text
content.read
    normalized readable resource

content.source_fields.read
    exact named source fields for one item

content.source_fields.collection
    bounded enumeration of source-auditable items

content.source_fields.update
    explicit provider-owned mutation path
```

Never infer write support from read support.

This distinction is important for AdSense historical-autotag audit/migration and may also be reused by Agent or administration tools.

---

## 13. Permission and security rules

Every capability must preserve provider ownership of permissions and business rules.

Rules:

- capability discovery must not broaden visibility;
- read services must be side-effect free;
- state changes must require normal provider ACL checks;
- state-changing web/admin paths require appropriate CSRF/token protection;
- public capability discovery must not imply authorization;
- sensitive or personal information should require separate permissions/scopes;
- one site's capability state must not leak into another site in multisite/shared-files deployments;
- service failures should be isolated so one provider cannot break the whole dashboard or capability catalogue.

---

## 14. Capability descriptors for machine adapters

A richer descriptor may optionally include:

```text
id
description
mode                read | write
service
version
stability
required_permission
input_schema
output_schema
risk
idempotent
personal_data
human_confirmation
```

JSON Schema-compatible input/output descriptions are preferred for new machine-facing capabilities.

This enables Agent or future adapters to map one Geeklog capability to:

- MCP;
- ChatGPT tools;
- REST/OpenAPI;
- administration interfaces;
- automation platforms;

without changing the owning plugin.

---

## 15. Compatibility and graceful fallback

Current modernization target remains:

- Geeklog 2.1.1 through 2.2.2;
- PHP 5.6 through PHP 8.1 unless a project explicitly broadens its tested range.

The capability contract must therefore:

- use PHP 5.6-compatible structures where required;
- feature-detect optional callbacks/functions;
- tolerate providers that implement only older Geeklog APIs;
- permit safe inference from existing Plugin APIs where explicit declaration is absent;
- never require direct SQL merely because a capability descriptor is missing.

Hub's audit may infer evidence from existing APIs, while explicit declarations should take precedence for capabilities that cannot be safely inferred.

---

## 16. Recommended implementation order

### P0 — shared memorandum contract

- maintain this document as the shared source of truth;
- keep capability identifiers stable;
- document additions before consumers hard-code conflicting names.

### P1 — current release candidates

Add or finalize capability declarations and `dashboard.summary` where practical for:

- Monitor 1.4;
- Documents 1.2;
- Videos 0.20;
- Maps 1.7;
- MediaGallery 1.8.

Do not destabilize an otherwise ready release merely to add nonessential optional capabilities; capability work must remain bounded and tested.

### P1 — Eclipse 1.2 — implemented

Eclipse 1.2 now:

- implements generic capability discovery through `plugin_getcapabilities_PLUGIN()`;
- consumes `dashboard.summary` through `PLG_invokeService()`;
- renders provider-owned metrics and management links without private-table access;
- promotes provider alerts plus conventional `pending` / `drafts` metrics into **Needs attention**;
- prefers structured dashboard metrics over legacy plugin statistics for the same provider to avoid duplicate presentation;
- degrades gracefully to native Geeklog plugin statistics for legacy providers;
- isolates invalid, unauthorized or unavailable provider summaries so one plugin cannot break the dashboard.

Documents 1.2 and Videos 0.20 are current reference implementations of the provider side of this contract.

### P1 — Hub

- make the interoperability audit understand the shared capability contract;
- expose Hub's own relationship/context/diagnostic capabilities;
- avoid a Hub-specific parallel registry.

### P2

- Forms 1.0: freeze `forms.list`, `forms.schema.read`, `dashboard.summary`;
- AdSense: dashboard and diagnostics;
- Contact: description/action capabilities when justified.

---

## 17. Architecture summary

```text
                    Geeklog Core
                        |
         +--------------+--------------+
         |              |              |
      Videos        Documents         Maps
         |              |              |
         +--------------+--------------+
                        |
             shared Geeklog contracts
             + capability declaration
                        |
          +-------------+-------------+
          |             |             |
         Hub          Agent        Eclipse
   context/relations  machine      dashboard
          |           access       presentation
          +-------------+
                        |
                 provider adapters
              MCP / ChatGPT / REST
```

The long-term objective is:

> **Plugins expose once; multiple consumers reuse the same contracts.**

A new plugin capability should become usable by Agent, Hub, Eclipse and future consumers without requiring each consumer to learn the provider's tables, private paths or business rules.
