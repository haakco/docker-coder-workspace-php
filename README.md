# docker-coder-workspace-php

Extends the Playwright workspace image with PHP build dependencies and the PHP
versions used by Coder projects. PHP is compiled in CI into `/opt/mise`, outside
the workspace home PVC, so a replacement workspace Pod can reuse the image's
installed versions.

## What's added on top of the base image

Build-time headers for common PHP core extensions (xml, openssl, curl, gd with avif/jpeg/png/webp/xpm, mbstring, zip, sqlite, readline, intl, bz2, tidy, xsl, sodium, pgsql, gmp, ldap) and for PECL extensions (`imagick`, `rdkafka`, `redis` with zstd). See the [Dockerfile](docker_build/Dockerfile) for the exact mapping.

## Who uses it

The Dockerfile bakes PHP 8.3.36 and 8.5.12 with the shared core and PECL extension
set. PHP 8.5 projects share the single 8.5.12 patch. awthy keeps 8.3.36 because it is a WordPress
project. A version missing from this image is compiled in every new workspace Pod
because `/opt/mise` is not a persistent volume.

PHP Coder workspaces include:

- **cb** (CourierBoost)
- **MacNan** (Laravel)
- **tlm** (TrackLab monorepo)

Their Coder workspace templates in `prod-infra/htz/germ/tf-ha/080_coder/templates/<project>/main.tf`
select this image. A project with an exact PHP pin can use an immutable digest so
its workspace always receives the build proven with that version.

## Rebuild cache

The Dockerfile uses the base image's `ccache` for PHP and PECL compilation.
Its `/php-ccache` BuildKit cache mount survives source-layer invalidation on the
same builder. The registry `:buildcache` remains the layer cache; it does not
export the compiler cache mount. An empty mount still yields a complete image.
CI forwards the runner's `CCACHE_REMOTE_STORAGE` into the build as an optional
build argument. When configured, ccache checks `/php-ccache` first, then uses the
shared HTTP cache so different builders and replacement builder Pods can reuse
compiler outputs. Each client keeps its own local cache; no client shares a
filesystem mount. Local builds default to local caching only. The endpoint is
private and contains no credentials; remote failures fall back to compilation.
This setting is build-only and is not baked into the workspace environment.
`CCACHE_NOHASHDIR=1` lets debug-symbol builds reuse results across the plugin's
random temporary directories; embedded debug paths may refer to an earlier
build directory.
The build logs print `ccache --show-stats` after each compile layer, so check
cacheable calls and hits before claiming a speedup. A changed PHP version or
compiler can still require a full compile.

Extension validation lists modules once per PHP version and checks the complete
required set against that output. A missing module or failed module-list command
still fails the build; command errors remain visible in the log.

Only the newly added PHP plugin, PHP installations and PHP download directory
need their ownership changed. Avoid recursively changing `/opt/mise` from the parent image: the
2026-10-02 rebuild spent roughly 7½ minutes in those broad passes.

## Adding a new PHP extension

1. Find the Debian/Ubuntu `-dev` package that ships the extension's headers (e.g. `pkg-config --list-all` inside a build).
2. Add it to the `apt-get install` line in `docker_build/Dockerfile`.
3. Update the comment block mapping `-dev` → extension.
4. Push — CI rebuilds `ghcr.io/haakco/coder-workspace-php:latest`.

When a project changes its exact PHP pin, add that version to the Dockerfile's
install step, `PHP_EXTENSION_VERSIONS`, and final extension check. After CI
publishes the image, update the consuming Coder template and prove a fresh
workspace selects the baked binary without a PHP source build.

## Tags

- `latest` — tracks `main` branch HEAD
- `edge` — same as `latest`
- `vX.Y.Z` — release tags (semver)
- Branch name — for PR builds
