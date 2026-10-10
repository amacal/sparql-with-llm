FROM mcr.microsoft.com/devcontainers/base:bookworm
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Deliberately no SPARQL engine, triple store, or RDF library baked in here —
# see CLAUDE.md's "No hardcoded technology". This image only provides generic
# infrastructure: enough to reach GitHub, to spin up and tear down whatever
# container a session's plan agent decides on (via Docker-outside-of-Docker),
# to run Claude Code itself, and to YAML-parse .history/.index files for
# validation (python3-yaml). Every engine/format choice is made fresh, per
# exercise, documented in that exercise's own notes/history — never here.
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl git ripgrep jq tzdata python3 python3-venv python3-yaml \
    && rm -rf /var/lib/apt/lists/*

RUN GH_RELEASE=$(curl -fsSL https://api.github.com/repos/cli/cli/releases/latest) \
    && GH_DEB=$(echo "$GH_RELEASE" | grep -o 'https://[^"]*linux_amd64\.deb' | head -1) \
    && if [ -n "$GH_DEB" ]; then \
         curl -fsSL "$GH_DEB" -o /tmp/gh.deb \
         && apt-get update && apt-get install -y /tmp/gh.deb \
         && rm -f /tmp/gh.deb; \
       else \
         GH_TAR=$(echo "$GH_RELEASE" | grep -o 'https://[^"]*linux_amd64\.tar\.gz' | head -1) \
         && curl -fsSL "$GH_TAR" -o /tmp/gh.tar.gz \
         && tar -xzf /tmp/gh.tar.gz -C /usr/local --strip-components=1 \
         && rm -f /tmp/gh.tar.gz; \
       fi \
    && rm -rf /var/lib/apt/lists/* \
    && gh --version

USER vscode
ENV PATH="/home/vscode/.local/bin:${PATH}"

RUN curl -fsSL https://claude.ai/install.sh | bash -s stable \
    && claude --version \
    && mkdir -p /home/vscode/.claude
