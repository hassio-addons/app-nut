ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:9.3.0
# hadolint ignore=DL3006
FROM ${BUILD_FROM} AS base

FROM base AS builder

ARG NUT_VERSION=2.8.4

# Setup builder
# hadolint ignore=DL3003
RUN \
    apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential libtool autoconf automake pkg-config \
        libusb-1.0-0-dev libsnmp-dev libssl-dev libxml2-dev libmodbus-dev libltdl-dev

RUN \
    mkdir -p /{app,build} \
    && cd /build \
    && NUT_FILENAME="nut-${NUT_VERSION}.tar.gz" \
    && NUT_DOWNLOAD_URL="https://github.com/networkupstools/nut/releases/download/v${NUT_VERSION}/${NUT_FILENAME}" \
    && curl -L -O "${NUT_DOWNLOAD_URL}" \
    && curl -L -O "${NUT_DOWNLOAD_URL}.sha256" \
    && sha256sum --check "${NUT_FILENAME}.sha256" \
    && tar -xzvf ${NUT_FILENAME} \
    && cd ${NUT_FILENAME%.tar.gz} \
    && ./configure --prefix=/usr --datadir=/usr/share/nut --sysconfdir=/etc/nut --with-drvpath=/usr/libexec/nut \
        --enable-static \
        --with-usb --with-modbus --with-snmp --with-nut-scanner --with-ssl \
    && make \
    && make DESTDIR=/app install


FROM base AS app

# Copy root filesystem
COPY rootfs /

# Copy nut files
COPY --from=builder /app /

# Setup app
# hadolint ignore=DL3003
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        usbutils=1:018-2 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# App environment

# Suppress warnings related to s6 not supporting lifecycle notifications
# Note: Unset this variable for debugging purposes
# ? Documentation: https://networkupstools.org/docs/man/nut.conf.html
ENV NUT_QUIET_INIT_UPSNOTIFY=true

# Build arguments
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_DESCRIPTION
ARG BUILD_NAME
ARG BUILD_REF
ARG BUILD_REPOSITORY
ARG BUILD_VERSION

# Labels
LABEL \
    io.hass.name="${BUILD_NAME}" \
    io.hass.description="${BUILD_DESCRIPTION}" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="app" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="Dale Higgs <dale3h@addons.community>" \
    org.opencontainers.image.title="${BUILD_NAME}" \
    org.opencontainers.image.description="${BUILD_DESCRIPTION}" \
    org.opencontainers.image.vendor="Home Assistant Community Apps" \
    org.opencontainers.image.authors="Dale Higgs <dale3h@addons.community>" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.url="https://frenck.dev/home-assistant-apps" \
    org.opencontainers.image.source="https://github.com/${BUILD_REPOSITORY}" \
    org.opencontainers.image.documentation="https://github.com/${BUILD_REPOSITORY}/blob/main/README.md" \
    org.opencontainers.image.created=${BUILD_DATE} \
    org.opencontainers.image.revision=${BUILD_REF} \
    org.opencontainers.image.version=${BUILD_VERSION}
