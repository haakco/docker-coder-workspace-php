# docker-coder-workspace-php

Extends the Playwright workspace image with PHP build dependencies and the PHP
versions used by Coder projects. PHP is compiled in CI into `/opt/mise`, outside
the workspace home PVC, so a replacement workspace Pod can reuse the image's
installed versions.

## What's added on top of the base image

Build-time headers for common PHP core extensions (xml, openssl, curl, gd with avif/jpeg/png/webp/xpm, mbstring, zip, sqlite, readline, intl, bz2, tidy, xsl, sodium, pgsql, gmp, ldap) and for PECL extensions (`imagick`, `rdkafka`, `redis` with zstd). See the [Dockerfile](docker_build/Dockerfile) for the exact mapping.

## Who uses it

The Dockerfile currently bakes PHP 8.3.31, 8.5.9, and 8.5.10 with the shared
core and PECL extension set. CB pins 8.5.10; Hosting and other PHP projects
still use 8.5.9. A version missing from this image is compiled in every new
workspace Pod because `/opt/mise` is not a persistent volume.

PHP Coder workspaces include:

- **cb** (CourierBoost)
- **MacNan** (Laravel)
- **tlm** (TrackLab monorepo)

Their Coder workspace templates in `prod-infra/htz/germ/tf-ha/080_coder/templates/<project>/main.tf` set `image = "ghcr.io/haakco/coder-workspace-php:latest"`.

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
