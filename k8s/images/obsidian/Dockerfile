# Based on https://github.com/sytone/obsidian-remote
# Copyright (c) 2022 Jon Bullen - MIT License
# See LICENSE.obsidian-remote for full license text

FROM ghcr.io/linuxserver/baseimage-kasmvnc:debianbookworm

ARG OBSIDIAN_VERSION=1.12.4

# Install system packages
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl libnss3 zlib1g-dev dbus-x11 uuid-runtime \
        libfuse2 libatk1.0-0 libatk-bridge2.0-0 libcups2 libgtk-3-0 \
        git && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /var/tmp/* /tmp/*

# Download and install Obsidian
ARG TARGETARCH
ARG FLAG=${TARGETARCH#amd64}
RUN curl -L -o ./obsidian.AppImage \
        "https://github.com/obsidianmd/obsidian-releases/releases/download/v${OBSIDIAN_VERSION}/Obsidian-${OBSIDIAN_VERSION}${FLAG:+-arm64}.AppImage" && \
    chmod +x ./obsidian.AppImage && \
    ./obsidian.AppImage --appimage-extract && \
    rm ./obsidian.AppImage

ENV CUSTOM_PORT="8080" \
    CUSTOM_HTTPS_PORT="8443" \
    CUSTOM_USER="" \
    PASSWORD="" \
    SUBFOLDER="" \
    TITLE="Obsidian v${OBSIDIAN_VERSION}" \
    FM_HOME="/vaults"

COPY root/ /

EXPOSE 8080 8443 3001 3443
VOLUME ["/config", "/vaults"]

HEALTHCHECK CMD curl --fail http://localhost:${CUSTOM_PORT}/ || exit 1
