# Geeklog Plugin Asset Loading and Cache Versioning

## Purpose

Plugin CSS and JavaScript are part of the shipped plugin contract. They must be packaged, loaded and cache-versioned deliberately so that administration and public pages do not keep stale assets after an install or upgrade.

This guide applies to modernized Geeklog plugins unless a plugin repository documents a stricter project-specific policy.

## Required baseline

For plugin-owned static CSS and JavaScript:

1. keep static assets in **external files** owned by the plugin;
2. load them through Geeklog's normal document header/footer integration;
3. load them **only on pages or contexts that need them** where practical;
4. append a deterministic cache-busting version to the asset URL;
5. package the asset in the installable archive and verify it in CI;
6. do not rely on a specific theme framework for core plugin administration.

Static plugin CSS should not normally be embedded as a large inline `<style>` block in a `.thtml` template or PHP page. Small computed inline fragments may occasionally be justified, but the normal rule for reusable/static styling is an external file.

The same principle applies to JavaScript: keep reusable/static code in external files rather than embedding large scripts in templates.

## Recommended Geeklog integration

For assets that belong in the document header, use the Plugin API where supported by the target compatibility range:

```php
function plugin_getheadercode_myplugin()
{
    // Return the required <link>, <script> or other header markup.
}
```

For administration-only assets, the callback should avoid loading those files globally. Check the current route/context and return an empty string on unrelated pages.

Example concept:

```php
function plugin_getheadercode_myplugin()
{
    global $_CONF;

    $script = isset($_SERVER['PHP_SELF'])
        ? str_replace('\\', '/', $_SERVER['PHP_SELF'])
        : '';

    if (strpos($script, '/admin/plugins/myplugin/') === false) {
        return '';
    }

    $url = rtrim($_CONF['site_admin_url'], '/')
        . '/plugins/myplugin/css/admin.css?v=1.2.0';

    return '<link rel="stylesheet" href="'
        . htmlspecialchars($url, ENT_QUOTES, 'UTF-8')
        . '" media="all" type="text/css">' . LB;
}
```

The exact path may differ by plugin layout. The important contract is conditional loading, external assets and cache versioning.

## Cache-busting/versioning

Every shipped CSS/JS URL should contain a cache-busting value that changes when the asset may have changed.

### Important Geeklog Resource caveat

Do **not** append a query string directly to a local filesystem-style asset path passed to `$_SCRIPTS->setCSSFile()` or `$_SCRIPTS->setJavaScriptFile()` on the 2.1.x/2.2.x compatibility range.

For example, avoid:

```php
$_SCRIPTS->setCSSFile(
    'myplugin',
    '/myplugin/css/style.css?v=1.2.0'
);
```

Geeklog's Resource loader validates local resources with an `is_file()` / readability check using the supplied path. A query string can therefore make the physical existence check fail, causing the asset not to be emitted at all.

Safe patterns are:

1. return a versioned `<link>` tag from `plugin_getheadercode_PLUGIN()` after resolving the real plugin/theme CSS path; or
2. register a fully qualified public URL (for example `https://example.com/myplugin/app.js?v=1.2.0`) when the Resource API path treats it as an external URL and therefore does not perform a local `is_file()` check; or
3. keep the native local-path loader unversioned when a specific helper requires a physical relative path, unless that helper is confirmed to support query strings.

Always verify the **rendered HTML** after an upgrade and confirm the expected CSS/JS tag is actually present. CI should also test the runtime loading strategy where practical, not merely grep for `?v=` in source code.

Acceptable strategies include:

- plugin release version:
  `admin.css?v=1.2.0`
- plugin release version plus asset modification timestamp:
  `admin.css?v=1.2.0-1791012345`
- another deterministic build/release identifier.

Using the plugin version is the minimum recommended baseline. Adding a file modification timestamp is useful during development and for projects where an asset may change without an immediate release-number change.

Do not use a random cache-buster on every request. That defeats browser caching instead of versioning it.

## Administration assets

Administration pages deserve the same asset discipline as public pages.

A modern admin implementation should normally:

- place admin-specific CSS under a clear plugin-owned path such as `admin/css/admin.css` or `css/admin.css`;
- load that stylesheet only for the plugin's administration routes;
- keep administration styling theme-neutral;
- avoid requiring UIKit, Bootstrap, Eclipse or another theme-specific framework;
- version the asset URL;
- keep significant page markup in templates where practical.

If the plugin has no custom administration CSS, it should rely on Geeklog/theme administration styles rather than add an empty stylesheet.

## Public assets

Public CSS/JS should also be external and versioned. Load it only where the plugin needs it when practical.

If the same asset is required on every public plugin page, a plugin-wide public-route check is appropriate. Do not load plugin assets on unrelated Geeklog pages without a real reason.

## Packaging and CI

The installable archive must contain every referenced asset.

Recommended build checks:

- verify the expected CSS/JS files exist in the ZIP;
- verify templates do not accidentally retain obsolete large inline `<style>` / `<script>` blocks after externalization;
- lint shipped PHP/INC files before packaging;
- fail the build when an asset referenced by runtime code is missing.

Example concept:

```sh
zipinfo -1 plugin.zip | grep -q '^myplugin/admin/css/admin.css$'
```

## Upgrade behavior

When an asset changes in a release:

- update the plugin version according to the project's release policy;
- ensure the URL cache-buster changes;
- verify the new archive contains the new asset;
- test after upgrade without manually clearing the browser cache.

If users must clear their browser cache to see a normal plugin upgrade, asset versioning is incomplete.

## Compatibility

Projects targeting Geeklog 2.1.1 through 2.2.2 must verify that their chosen asset-loading mechanism works across that range and provide a narrow fallback where required.

A project explicitly targeting only Geeklog 2.2.2 may use the 2.2.2-compatible path directly.

## Acceptance checklist

- [ ] static plugin CSS is stored in external files
- [ ] static plugin JavaScript is stored in external files
- [ ] admin-only assets are not loaded globally
- [ ] public-only assets are not loaded globally without need
- [ ] CSS/JS URLs contain deterministic cache versions
- [ ] asset version changes when shipped content changes
- [ ] installable archive contains every referenced asset
- [ ] CI verifies important packaged assets
- [ ] templates do not retain obsolete large inline asset blocks
- [ ] plugin administration remains usable without a theme-specific framework
- [ ] upgrade test works without manual browser cache clearing

## Guiding principle

> A plugin asset is not correctly shipped merely because the file exists: it must be externalized where appropriate, loaded in the correct context, versioned for cache invalidation, and verified in the installable archive.
