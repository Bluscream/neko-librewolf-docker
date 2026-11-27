# neko with LibreWolf instead of Firefox (neko ships no LibreWolf image), built like neko's own
# Firefox app on neko's base image. Every extension is allowed; uBlock Origin, Dark Reader and FoxMCP
# are force-installed, and the FoxMCP bridge runs under supervisord so MCP agents can drive
# this browser (container port 8771).
ARG BASE_IMAGE=ghcr.io/m1k1o/neko/base:latest
FROM $BASE_IMAGE
ARG FOXMCP_VERSION=1.7.0

RUN set -eux; apt-get update; \
    apt-get install -y --no-install-recommends openbox wget gnupg ca-certificates python3-venv socat; \
    wget -qO- https://repo.librewolf.net/pubkey.gpg | gpg --dearmor -o /usr/share/keyrings/librewolf.gpg; \
    printf 'Types: deb\nURIs: https://repo.librewolf.net\nSuites: librewolf\nComponents: main\nArchitectures: %s\nSigned-By: /usr/share/keyrings/librewolf.gpg\n' \
      "$(dpkg --print-architecture)" > /etc/apt/sources.list.d/librewolf.sources; \
    apt-get update; \
    apt-get install -y --no-install-recommends librewolf; \
    python3 -m venv /opt/foxmcp; \
    /opt/foxmcp/bin/pip install --no-cache-dir "foxmcp==${FOXMCP_VERSION}"; \
    mkdir -p /home/neko/.librewolf/profile.default/extensions; \
    apt-get --purge autoremove -y gnupg; \
    apt-get clean -y; \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/*

COPY supervisord.conf /etc/neko/supervisord/librewolf.conf
COPY openbox.xml /etc/neko/openbox.xml
COPY policies.json /usr/share/librewolf/distribution/policies.json
COPY --chown=neko profiles.ini /home/neko/.librewolf/profiles.ini
COPY --chown=neko librewolf.overrides.cfg /home/neko/.librewolf/librewolf.overrides.cfg
RUN chown -R neko:neko /home/neko/.librewolf
