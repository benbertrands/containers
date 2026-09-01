# obsidian-headless: the headless client for Obsidian Sync.
#
# There is no upstream container image -- obsidian-headless ships as an npm
# package and nothing else -- so this repository builds one. The point is that a
# deployment can pin a version rather than resolving one at deploy time, where
# `npm install -g` would leave the installed version as whatever npm happened to
# serve that day and offer nothing for Renovate to open a pull request against.
#
# Two pins, two Renovate managers:
#   * this FROM line, handled by Renovate's built-in dockerfile manager
#   * OBSIDIAN_HEADLESS_VERSION below, handled by the custom npm manager in
#     renovate.json
# The image tag pushed to ghcr.io is exactly OBSIDIAN_HEADLESS_VERSION, derived
# by .github/workflows/images.yml.
#
# slim, which works only because every native addon here arrives prebuilt. The
# package's own btime addon ships Windows and macOS binaries and is inert on
# Linux, but its better-sqlite3 dependency IS a Linux native addon: it is
# installed via prebuild-install, which downloads a prebuilt binary rather than
# compiling. If a matching prebuild ever stops being published for this Node
# version, npm falls back to `node-gyp rebuild` and the build breaks here, since
# slim ships neither python3 nor a compiler. Adding build-essential at that point
# is the fix; note the downloaded binary is fetched outside npm's integrity
# checks, so it is covered by neither the version pin below nor Renovate.
FROM docker.io/library/node:22-trixie-slim

# renovate: datasource=npm depName=obsidian-headless
ARG OBSIDIAN_HEADLESS_VERSION=0.0.14

LABEL org.opencontainers.image.source="https://github.com/benbertrands/containers" \
      org.opencontainers.image.description="Headless client for Obsidian Sync (obsidian-headless ${OBSIDIAN_HEADLESS_VERSION})" \
      org.opencontainers.image.licenses="MIT"

# --omit=dev and no npm cache: this layer is the whole image, and a cache that is
# never read again is just bytes every consumer has to pull.
RUN npm install -g --omit=dev "obsidian-headless@${OBSIDIAN_HEADLESS_VERSION}" \
    && npm cache clean --force

# `ob` resolves its config directory on Linux as $XDG_CONFIG_HOME/obsidian-headless,
# falling back to ~/.config. It is set explicitly so persistent state lives at one
# predictable path rather than depending on the image's home directory layout.
#
# /config is created and handed to the runtime user here because the container
# cannot create it itself: it runs as a non-root user and / is root-owned. Doing
# it at build time means the image also works when nothing is mounted over it.
ENV XDG_CONFIG_HOME=/config
RUN mkdir -p /config && chown node:node /config

# The node image already provides a `node` user at uid 1000. Running as that user
# rather than root means a rootless runtime mapping the invoking user onto uid
# 1000 leaves anything written into a mounted volume owned by that user on the
# host.
ENV HOME=/home/node \
    NODE_ENV=production
USER node

# `ob` itself, so a caller only has to supply the subcommand and its flags.
ENTRYPOINT ["ob"]
CMD ["--help"]
