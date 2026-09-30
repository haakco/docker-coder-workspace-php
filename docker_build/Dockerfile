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
# pecl_http, not http: the plugin builds the download URL from the name it is given as
# https://pecl.php.net/get/<name>, and the PECL package is called pecl_http. Asking for
# 'http' returns a 404 with "package 'http' does not exist", which the plugin reports only
# as "tar: Error is not recoverable" after a full source compile. The installed extension
# is still http.so - the name comes from the package itself - so the workspace sees the
# same extension either way. cb's own .mise/install-php-extensions.sh maps the two the
# same way, module http <- PECL package pecl_http.
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
    # pecl_extensions is deliberately NOT set here. As a mise config option it applies to
    # every version, and the plugin refuses its PECL path on PHP 8.5 or newer with "PECL is
    # not available for PHP 8.5.5 in this plugin", because 8.5 no longer ships the pecl
    # binary. The list is passed as an environment variable to the one install that can use
    # it instead - see the php@8.3.31 layer - and 8.5 gets the same extensions built from
    # source in the layer after it.
    && post_install="$MISE_DATA_DIR/plugins/php/hooks/post_install.lua" \
    && [ -f "$post_install" ] \
    && if ! grep -q HAAKCO_PIE_BEST_EFFORT "$post_install"; then \
         printf '\n-- HAAKCO_PIE_BEST_EFFORT: PIE setup is best-effort. Verification is\n-- unreliable on this image and no project requests PIE extensions, so a flaky\n-- check must not fail the install and trigger a rebuild on every start.\nlocal haakco_install_pie = php_packages.install_pie\nphp_packages.install_pie = function(sdkPath, version)\n  haakco_install_pie(sdkPath, version)\n  return true\nend\n' >> "$post_install"; \
       fi \
    && chown -R 1000:1000 /opt/mise

RUN --mount=type=secret,id=github_token \
    export GITHUB_TOKEN="$(cat /run/secrets/github_token 2>/dev/null || true)" \
    && MISE_DATA_DIR=/opt/mise MISE_YES=1 PHP_VERBOSE=1 mise install php@8.3.31

# Bake the versions PHP workspaces pin: 8.3.31 for the WordPress project (awthy),
# 8.5.10 for every other PHP project. One 8.5 patch is deliberate - two of them were two
# near-identical source builds to compile, verify and ship, and a project could only be
# on whichever one the image happened to carry. A project that moves onto a different
# version needs a matching install and extension check below.
#
# These versions do not use the plugin's PECL path. It cannot build PECL extensions on 8.5 at
# all, and it installs the http extension under its PECL package name, so the extension
# never loads as http - which is why the verification below caught http missing. Every
# extension is built from source, for each version, in the step below.
RUN --mount=type=secret,id=github_token \
    export GITHUB_TOKEN="$(cat /run/secrets/github_token 2>/dev/null || true)" \
    && MISE_DATA_DIR=/opt/mise MISE_YES=1 PHP_VERBOSE=1 mise install php@8.5.10

ARG PHP_EXTENSION_VERSIONS="8.3.31 8.5.10"
ARG MISE_PHP_PECL_EXTENSIONS="igbinary msgpack redis imagick rdkafka raphf pecl_http pcov xdebug opentelemetry"

# Build the extensions from source and record what each one produced.
#
# The downloads sit in a cache mount and are revalidated with `curl -z`, so a rebuilt layer
# reuses a tarball the server says is unchanged rather than re-fetching every PECL release -
# and never silently installs an older one, because a changed release is downloaded again.
# The build tree stays a fresh mktemp so a rebuilt layer cannot pick up earlier objects.
#
# The ini files are deliberately NOT written here - see the layer after this one.
RUN --mount=type=cache,target=/pecl-cache \
    set -eu \
    && : > /tmp/php-extensions.tsv \
    && for version in ${PHP_EXTENSION_VERSIONS}; do \
         prefix="/opt/mise/installs/php/${version}"; \
         extdir="$("${prefix}/bin/php-config" --extension-dir)"; \
         mkdir -p "${extdir}"; \
         for ext in ${MISE_PHP_PECL_EXTENSIONS}; do \
           work="$(mktemp -d)"; \
           before="$(mktemp)"; after="$(mktemp)"; \
           echo "building ${ext} for php@${version}"; \
           tarball="/pecl-cache/${ext}.tgz"; \
           curl -fsSL --retry 5 --retry-delay 5 --retry-all-errors -z "${tarball}" \
             "https://pecl.php.net/get/${ext}" -o "${tarball}" \
             || { echo "could not download ${ext} for php@${version}"; exit 1; }; \
           tar xzf "${tarball}" -C "${work}" --strip-components=1; \
           extra_args=""; \
           if [ "${ext}" = "xdebug" ]; then extra_args="--enable-xdebug"; fi; \
           ls "${extdir}" 2>/dev/null | sort > "${before}" || true; \
           ( cd "${work}" \
             && "${prefix}/bin/phpize" \
             && ./configure ${extra_args} --with-php-config="${prefix}/bin/php-config" \
             && make -j"$(nproc)" \
             && make install ); \
           ls "${extdir}" 2>/dev/null | sort > "${after}" || true; \
           so="$(comm -13 "${before}" "${after}" | grep '\.so$' | head -1)"; \
           rm -rf "${work}" "${before}" "${after}"; \
           [ -n "${so}" ] \
             || { echo "building ${ext} for php@${version} produced no extension"; exit 1; }; \
           printf '%s\t%s\t%s\n' "${version}" "${ext}" "${so}" >> /tmp/php-extensions.tsv; \
           echo "  built ${so} for php@${version}"; \
         done; \
       done

# Write each extension's ini file from the build manifest.
#
# This is its own layer so that an ini change is seconds, not a recompile: the xdebug
# load-key fix invalidated the build layer above and rebuilt thirty extensions because
# the ini write lived inside that loop. The two-digit prefix is the build order, which is
# what the existing files were named after.
RUN set -eu \
    && for version in ${PHP_EXTENSION_VERSIONS}; do \
         prefix="/opt/mise/installs/php/${version}"; \
         scandir="${prefix}/conf.d"; \
         mkdir -p "${scandir}"; \
         index=0; \
         for ext in ${MISE_PHP_PECL_EXTENSIONS}; do \
           index=$((index + 1)); \
           so="$(awk -F'\t' -v v="${version}" -v e="${ext}" '$1 == v && $2 == e { print $3 }' /tmp/php-extensions.tsv)"; \
           [ -n "${so}" ] \
             || { echo "the build step recorded no extension for ${ext} on php@${version}"; exit 1; }; \
           ini_key="extension"; \
           # Xdebug only loads as a Zend extension. Written as `extension=` it prints
           # "Xdebug MUST be loaded as a Zend extension" on every php start and is
           # silently not loaded, so the project's own extension check cannot tell the
           # difference - the assertion below is what catches a regression.
           if [ "${ext}" = "xdebug" ]; then ini_key="zend_extension"; fi; \
           printf '%s\n' "${ini_key}=${so}" \
             > "${scandir}/$(printf '%02d' "${index}")-${so%.so}.ini"; \
           echo "  enabled ${so} for php@${version} (${ini_key})"; \
         done; \
       done

# Prove the compile-time extensions the projects verify are actually present. mise
# reports a tool as installed without rechecking how it was built, so a missing
# flag here would otherwise only surface as a failed workspace start.
RUN MISE_DATA_DIR=/opt/mise mise reshim \
    && chown -R 1000:1000 /opt/mise \
    && MISE_DATA_DIR=/opt/mise mise ls php \
    && core_extensions="ldap xsl sodium zip" \
    && pecl_extensions="igbinary msgpack redis imagick rdkafka raphf http pcov xdebug opentelemetry" \
    && for version in ${PHP_EXTENSION_VERSIONS}; do \
         for ext in ${core_extensions} ${pecl_extensions}; do \
           MISE_DATA_DIR=/opt/mise mise exec "php@${version}" -- php -m 2>/dev/null \
             | grep -qix "$ext" \
             || { echo "php@${version} is missing the $ext extension"; exit 1; }; \
         done; \
       done \
    && for version in ${PHP_EXTENSION_VERSIONS}; do \
         # `php -m` lists xdebug either way, so the extension check above passes even
         # when it is loaded as a normal extension. Fail the build instead of shipping a
         # warning on every php start.
         if grep -rqs '^extension=xdebug' "/opt/mise/installs/php/${version}/conf.d/"; then \
           echo "php@${version} loads xdebug as a normal extension; it needs zend_extension"; \
           exit 1; \
         fi; \
       done \
    && echo "php@${PHP_EXTENSION_VERSIONS} carry the core and PECL extension sets"

USER coder
WORKDIR /home/coder

# Smoke assertions, moved out of CI and into the build so they run on the shared BuildKit
# for both the PR path and the publish path: an image can no longer publish without them.
# Verbatim from the retired `Smoke test` workflow step, including its `set -e` - without
# that, the trailing echo would mask any earlier failure. Runs as `coder`, the user
# `docker run` used, so PATH and tool behaviour match what was validated there.
RUN <<'EOF'
set -e
node --version
corepack --version
pi --version
dpkg -s libgd3 >/dev/null
pkg-config --exists libxml-2.0
test -d /ms-playwright
test -f /etc/coder-skel/.pi/agent/settings.json
echo SMOKE_OK
EOF

ENTRYPOINT ["/usr/local/bin/workspace-entrypoint.sh"]
