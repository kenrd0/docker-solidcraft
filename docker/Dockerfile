# syntax=docker/dockerfile:1

FROM ghcr.io/linuxserver/baseimage-selkies:debiantrixie

# Upstream release version, without the leading v.
ARG SOLIDCRAFT_VERSION=0.30.0
ARG TARGETARCH

LABEL org.opencontainers.image.title="SolidCraft" \
    org.opencontainers.image.description="SolidCraft desktop streamed with Selkies" \
    org.opencontainers.image.version="${SOLIDCRAFT_VERSION}"

ENV TITLE=SolidCraft \
    NO_GAMEPAD=true \
    PIXELFLUX_WAYLAND=true

# SolidCraft's rfd file dialogs fall back to zenity when no XDG portal is running.
RUN set -eu; \
    echo "**** download SolidCraft ****"; \
    case "${TARGETARCH}" in \
      amd64) app_arch=x86_64 ;; \
      arm64) app_arch=aarch64 ;; \
      *) echo "Unsupported architecture: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    asset="solidcraft-linux-${app_arch}.deb"; \
    release_url="https://github.com/tangency-labs/solidcraft/releases/download/v${SOLIDCRAFT_VERSION}"; \
    curl -fSL --retry 3 "${release_url}/${asset}" -o "/tmp/${asset}"; \
    curl -fSL --retry 3 "${release_url}/SHA256SUMS.txt" -o /tmp/SHA256SUMS.txt; \
    cd /tmp; \
    awk -v asset="${asset}" '$2 == asset { print }' SHA256SUMS.txt > package.sha256; \
    test "$(wc -l < package.sha256)" -eq 1; \
    sha256sum -c package.sha256; \
    echo "**** install SolidCraft and runtime dependencies ****"; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      libegl1 \
      libvulkan1 \
      mesa-vulkan-drivers \
      zenity \
      "/tmp/${asset}"; \
    solidcraft-cli --version; \
    echo "**** cleanup ****"; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/* /var/tmp/* /tmp/*

COPY root/ /
RUN chmod +x /defaults/autostart /defaults/autostart_wayland

EXPOSE 3001
VOLUME /config
