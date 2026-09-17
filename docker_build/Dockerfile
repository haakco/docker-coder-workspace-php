# syntax=docker/dockerfile:1

# =============================================================================
# Coder workspace image with PHP build dependencies.
# Extends ghcr.io/haakco/coder-workspace-playwright (which itself extends
# coder-workspace). Inheriting from -playwright means PHP projects also get
# Chromium/Firefox/WebKit baked in for E2E browser testing without each
# template having to opt in. Shared by projects that use mise's PHP plugin
# to compile PHP from source: cb (CourierBoost), MacNan, tlm.
#
# Each -dev header maps to one or more PHP extensions (core or PECL):
#
#   Core (enabled via PHP_CONFIGURE_OPTIONS):
#     libxml2-dev          → xml, soap, dom, simplexml
#     libssl-dev           → openssl
#     libcurl4-openssl-dev → curl
#     libavif/jpeg/png/webp/xpm-dev + libfreetype6-dev → gd
#     libonig-dev          → mbstring
#     libzip-dev           → zip
#     libsqlite3-dev       → sqlite3, pdo_sqlite
#     libreadline-dev      → readline
#     libicu-dev           → intl
#     libbz2-dev           → bz2
#     libtidy-dev          → tidy
#     libxslt1-dev         → xsl
#     libsodium-dev        → sodium
#     libpq-dev            → pgsql, pdo_pgsql
#     libgmp-dev           → gmp
#     libldap2-dev         → ldap
#
#   PECL:
#     libmagickwand-dev    → imagick
#     librdkafka-dev       → rdkafka
#     libzstd-dev          → redis (optional zstd compression)
#
# Add more -dev packages here if PHP's ./configure starts failing for an
# extension we haven't covered yet.
# =============================================================================
ARG BASE_IMAGE_NAME=ghcr.io/haakco/coder-workspace-playwright
ARG BASE_IMAGE_VERSION=latest

FROM ${BASE_IMAGE_NAME}:${BASE_IMAGE_VERSION}

USER root

RUN apt-get update && apt-get install -y --no-install-recommends \
    libxml2-dev libssl-dev libcurl4-openssl-dev \
    libavif-dev libjpeg-dev libpng-dev libwebp-dev libxpm-dev libonig-dev \
    libzip-dev libsqlite3-dev libreadline-dev \
    libicu-dev libfreetype6-dev libbz2-dev \
    libtidy-dev libxslt1-dev libsodium-dev \
    libpq-dev libgmp-dev libldap2-dev \
    libmagickwand-dev librdkafka-dev libzstd-dev \
    # Runtime GD library. PHP projects previously installed this on every
    # workspace start; baking it in keeps workspace startup from running
    # apt-get update each boot.
    libgd3 \
    && rm -rf /var/lib/apt/lists/*

# -----------------------------------------------------------------------------
# PHP, compiled once here instead of in every workspace.
#
# Measured 2026-09-16 on a cold workspace: compiling PHP from source takes 804s -
# about half of a 1648s start - and every PHP workspace paid it again, because the
# toolchain lives on an ephemeral volume. The base image now bakes the shared
# toolchain at /opt/mise, outside the /home/coder PVC mount that would otherwise
# hide anything under the home directory; this image adds PHP to it, so one build
# serves every workspace of every PHP project.
#
# The versions are the pins the estate's PHP projects actually use, listed once: a
# bump is now a one-line change plus a rebuild, rather than a fresh source compile
# on every workspace start. Each version is its own layer so that bumping one does
# not discard the BuildKit cache for the others - a PHP compile is ~13 minutes.
#
# mise treats an installed tool as installed without rechecking how it was built,
# so these build settings must match what the projects expect.
#
# The extension set goes through the plugin's own extra_configure_options option,
# not through the environment. The plugin's mise_env hook does
#   set_env(env_vars, "PHP_EXTRA_CONFIGURE_OPTIONS", options.get(ctx, "extra_configure_options"))
# whenever that option is set, so a project's own PHP_EXTRA_CONFIGURE_OPTIONS - and
# anything exported here - is overwritten by it. An earlier revision exported the
# full set and set the option to just --with-sodium, so the option won and every
# baked PHP was missing ldap and xsl; the build's own assertion caught it.
#
# The options are those the projects supply in their own mise.toml, which a
# workspace build reads and this image build cannot: the union across the PHP
# projects is ldap, pgsql, sodium, xsl and zip (cb needs ldap/sodium/xsl/zip, tlm
# adds pgsql), and it is exactly what the -dev packages above provide.
#
# A project needing different flags is caught rather than silently wrong: mise
# will not recompile a baked tool, and each PHP project's mise enter hook verifies
# the extension set on every workspace start.
#
# PECL extensions are baked for the same reason. They are compile-time too - the
# plugin installs them through its own pecl_extensions option during the PHP
# build - and without them cb's composer install fails with 'it is missing from
# your system' for ext-imagick and ext-opentelemetry, leaving vendor/ absent and
# the workspace reporting a start error. The union of the PHP projects' PECL lists
# is igbinary, msgpack, redis, imagick, rdkafka, raphf, http, pcov, xdebug and
# opentelemetry, and the -dev packages above cover all of them.
#
#   PHP_EXTRA_CONFIGURE_OPTIONS  the extension set above, set through the plugin's
#                           extra_configure_options option
#   --with-sodium        sodium extension, set through the plugin's configure hook
#   HAAKCO_PIE_BEST_EFFORT  PIE verification is unreliable here and no project asks
#                           for PIE extensions, so a flaky check must not fail the
#                           install and trigger a rebuild on every workspace start.
#
# Every mise call here sets MISE_DATA_DIR. Without it mise uses its default data
# directory, and the plugin lands somewhere the later layers never look: the first
# version of this layer installed the plugin into ~/.local/share/mise, so
# /opt/mise/plugins stayed empty, the next RUN silently auto-installed mise's
# built-in jdx/vfox-php backend instead, and the build failed on a missing libgd
# because that backend does not install build dependencies. The PIE patch was
# skipped by its own [ -f ] guard for the same reason, which is why the guard is
# now an assertion.
#
# `mise config set -f` needs its file to exist. In a workspace it does, because the
# dotfiles config is already symlinked there before the startup script runs;
# nothing has created it at this point in the build, so it is touched first.
#
# The PIE patch is written with printf rather than a heredoc on purpose: a heredoc
# inside a RUN ends the instruction, so anything after its terminator is parsed as
# a new Dockerfile instruction and the build fails with "unknown instruction".
# -----------------------------------------------------------------------------
ARG MISE_PHP_PLUGIN="https://github.com/verzly/mise-php#latest"
ARG PHP_EXTRA_CONFIGURE_OPTIONS="--with-ldap --with-pgsql --with-sodium --with-xsl --with-zip"
ARG PHP_PECL_EXTENSIONS="igbinary msgpack redis imagick rdkafka raphf http pcov xdebug opentelemetry"
RUN --mount=type=secret,id=github_token \
    set -eu \
    && export GITHUB_TOKEN="$(cat /run/secrets/github_token 2>/dev/null || true)" \
    && export MISE_DATA_DIR=/opt/mise \
    && mkdir -p "$MISE_DATA_DIR/plugins" /root/.config/mise \
    && mise plugin install --force php "${MISE_PHP_PLUGIN}" \
    # Fail loudly if the plugin is not the one this build depends on. mise happily
    # falls back to its built-in backend, which compiles PHP but ships no build
    # dependencies, and the only symptom is an unrelated configure error later.
    && mise plugins ls --urls | grep -q 'verzly/mise-php' \
    && touch /root/.config/mise/config.toml \
    && mise config set -f /root/.config/mise/config.toml \
         env._.php.extra_configure_options --type string -- "${PHP_EXTRA_CONFIGURE_OPTIONS}" \
    && mise config set -f /root/.config/mise/config.toml \
         env._.php.pecl_extensions --type string -- "${PHP_PECL_EXTENSIONS}" \
    && post_install="$MISE_DATA_DIR/plugins/php/hooks/post_install.lua" \
    && [ -f "$post_install" ] \
    && if ! grep -q HAAKCO_PIE_BEST_EFFORT "$post_install"; then \
         printf '\n-- HAAKCO_PIE_BEST_EFFORT: PIE setup is best-effort. Verification is\n-- unreliable on this image and no project requests PIE extensions, so a flaky\n-- check must not fail the install and trigger a rebuild on every start.\nlocal haakco_install_pie = php_packages.install_pie\nphp_packages.install_pie = function(sdkPath, version)\n  haakco_install_pie(sdkPath, version)\n  return true\nend\n' >> "$post_install"; \
       fi \
    && chown -R 1000:1000 /opt/mise

RUN --mount=type=secret,id=github_token \
    export GITHUB_TOKEN="$(cat /run/secrets/github_token 2>/dev/null || true)" \
    && MISE_DATA_DIR=/opt/mise MISE_YES=1 PHP_VERBOSE=1 mise install php@8.3.31
RUN --mount=type=secret,id=github_token \
    export GITHUB_TOKEN="$(cat /run/secrets/github_token 2>/dev/null || true)" \
    && MISE_DATA_DIR=/opt/mise MISE_YES=1 PHP_VERBOSE=1 mise install php@8.5.5
RUN --mount=type=secret,id=github_token \
    export GITHUB_TOKEN="$(cat /run/secrets/github_token 2>/dev/null || true)" \
    && MISE_DATA_DIR=/opt/mise MISE_YES=1 PHP_VERBOSE=1 mise install php@8.5.7
RUN --mount=type=secret,id=github_token \
    export GITHUB_TOKEN="$(cat /run/secrets/github_token 2>/dev/null || true)" \
    && MISE_DATA_DIR=/opt/mise MISE_YES=1 PHP_VERBOSE=1 mise install php@8.5.9

# Prove the compile-time extensions the projects verify are actually present. mise
# reports a tool as installed without rechecking how it was built, so a missing
# flag here would otherwise only surface as a failed workspace start.
RUN MISE_DATA_DIR=/opt/mise mise reshim \
    && chown -R 1000:1000 /opt/mise \
    && MISE_DATA_DIR=/opt/mise mise ls php \
    && for version in 8.3.31 8.5.5 8.5.7 8.5.9; do \
         for ext in ldap xsl sodium zip \
                    igbinary msgpack redis imagick rdkafka raphf http pcov xdebug opentelemetry; do \
           MISE_DATA_DIR=/opt/mise mise exec "php@${version}" -- php -m 2>/dev/null \
             | grep -qix "$ext" \
             || { echo "php@${version} is missing the $ext extension"; exit 1; }; \
         done; \
       done \
    && echo "all four PHP builds carry the core and PECL extension sets"

USER coder
WORKDIR /home/coder
ENTRYPOINT ["/usr/local/bin/workspace-entrypoint.sh"]
