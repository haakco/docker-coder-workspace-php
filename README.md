# docker-coder-workspace-php

Extends [`ghcr.io/haakco/coder-workspace`](https://github.com/haakco/docker-coder-workspace) with the system `-dev` headers needed by `mise`'s PHP plugin to compile PHP from source.

## What's added on top of the base image

Build-time headers for common PHP core extensions (xml, openssl, curl, gd, mbstring, zip, sqlite, readline, intl, bz2, tidy, xsl, sodium, pgsql, gmp, ldap) and for PECL extensions (`imagick`, `rdkafka`, `redis` with zstd). See the [Dockerfile](docker_build/Dockerfile) for the exact mapping.

## Who uses it

Projects whose `.mise.toml` contains `php = { version = "8.4.x", ... }` with a build-from-source PHP plugin:

- **cb** (CourierBoost)
- **MacNan** (Laravel)
- **tlm** (TrackLab monorepo)

Their Coder workspace templates in `prod-infra/htz/germ/tf-ha/080_coder/templates/<project>/main.tf` set `image = "ghcr.io/haakco/coder-workspace-php:latest"`.

## Adding a new PHP extension

1. Find the Debian/Ubuntu `-dev` package that ships the extension's headers (e.g. `pkg-config --list-all` inside a build).
2. Add it to the `apt-get install` line in `docker_build/Dockerfile`.
3. Update the comment block mapping `-dev` → extension.
4. Push — CI rebuilds `ghcr.io/haakco/coder-workspace-php:latest`.

## Tags

- `latest` — tracks `main` branch HEAD
- `edge` — same as `latest`
- `vX.Y.Z` — release tags (semver)
- Branch name — for PR builds
