# Claude Code, packaged as a container image.
#
# Anthropic ships a devcontainer *feature* and a reference .devcontainer, both
# described upstream as a working example rather than a maintained base image, so
# there is no official image to pin. This image exists for the same reason as the
# obsidian-headless one beside it: a deployment can pin a version rather than
# resolving one at deploy time, which `npm install -g` can never offer.
#
# Renovate manages both pins -- the FROM line via its dockerfile manager, and
# CLAUDE_CODE_VERSION via the custom npm manager in renovate.json. The image tag
# pushed to ghcr.io is exactly CLAUDE_CODE_VERSION.
FROM docker.io/library/node:24-trixie-slim

# renovate: datasource=npm depName=@anthropic-ai/claude-code
ARG CLAUDE_CODE_VERSION=2.1.291

LABEL org.opencontainers.image.source="https://github.com/benbertrands/containers" \
      org.opencontainers.image.description="Claude Code ${CLAUDE_CODE_VERSION}" \
      org.opencontainers.image.licenses="MIT"

# git is not optional here: it is what lets Claude Code work inside a repository
# mounted into the container, and what `--spawn worktree` needs if that mode is
# ever turned on. ca-certificates is needed to verify api.anthropic.com; the slim
# base image does not ship it.
#
# The node image ships ripgrep's dependencies but not ripgrep itself; Claude Code
# bundles its own, so it is deliberately not installed here.
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates git \
    && rm -rf /var/lib/apt/lists/*

# This package runs a postinstall script, which executes as root at build time --
# before the USER drop below. That is inherent to installing it at all; the
# mitigation is the release-age delay in renovate.json, which keeps a freshly
# compromised release from being picked up the day it appears.
#
# That postinstall is also what makes the install work: it puts the native binary
# over bin/claude.exe, which ships as a stub that only prints an error and exits
# 1. npm 11 warns that the script is "not yet covered by allowScripts" but still
# runs it; --allow-scripts names it explicitly, so a later npm that stops running
# uncovered scripts by default does not silently leave the stub in place.
#
# The version check is the proof: the stub fails it, and so does an install that
# resolved to anything other than the pinned version. It runs here rather than
# as a CI step after the build, because the workflow pushes in the same step it
# builds in -- a failing RUN is what keeps a broken image off ghcr.io.
RUN npm install -g --allow-scripts=@anthropic-ai/claude-code \
        "@anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}" \
    && npm cache clean --force \
    && installed="$(claude --version)" \
    && case "$installed" in \
         "${CLAUDE_CODE_VERSION} "*) ;; \
         *) echo "claude --version printed '$installed', expected ${CLAUDE_CODE_VERSION}" >&2; exit 1 ;; \
       esac

# The bundled updater would replace the pinned version at runtime, which defeats
# the point of pinning it: the image tag would stop describing what is actually
# running. This is not one of the variables that disable Remote Control.
#
# It also means the container never self-patches, so keeping up with upstream is
# entirely a matter of the Renovate cadence plus a redeploy.
ENV DISABLE_AUTOUPDATER=1

# The node image already provides a `node` user at uid 1000. Running as that user
# rather than root means a rootless runtime mapping the invoking user onto uid
# 1000 leaves anything written into a mounted volume owned by that user on the
# host.
#
# HOME holds the login, the workspace-trust record and ~/.claude.json, all of
# which have to survive a restart -- mount a volume here.
ENV HOME=/home/node
USER node

# No CMD: running the image with no arguments starts an ordinary interactive
# Claude Code session, which is what the one-time bootstrap (workspace trust,
# then /login) needs. Pass `remote-control` and its flags as arguments to run it
# in server mode instead.
ENTRYPOINT ["claude"]
