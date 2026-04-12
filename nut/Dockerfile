ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:9.3.0
# hadolint ignore=DL3006
FROM ${BUILD_FROM} AS base

FROM base AS builder

ARG NUT_VERSION=2.8.5

# Setup builder
RUN \
    apt-get update \
    && apt-get install -y --no-install-recommends \
        autoconf=2.72-3.1 \
        automake=1:1.17-4 \
        build-essential=12.12 \
        libhidapi-dev=0.14.0-1+b2 \
        libltdl-dev=2.5.4-4 \
        libmodbus-dev=3.1.11-2 \
        libsnmp-dev=5.9.4+dfsg-2+deb13u1 \
        libssl-dev=3.5.5-1~deb13u2 \
        libusb-1.0-0-dev=2:1.0.28-1 \
        libxml2-dev=2.12.7+dfsg+really2.9.14-2.1+deb13u2 \
        libtool=2.5.4-4 \
        pkg-config=1.8.1-4 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# hadolint ignore=DL3003
RUN \
    mkdir -p /app /build \
    && cd /build \
    && NUT_FILENAME="nut-${NUT_VERSION}.tar.gz" \
    && NUT_DOWNLOAD_URL="https://github.com/networkupstools/nut/releases/download/v${NUT_VERSION}/${NUT_FILENAME}" \
    && curl -L -O "${NUT_DOWNLOAD_URL}" \
    && curl -L -O "${NUT_DOWNLOAD_URL}.sha256" \
    && sha256sum --check "${NUT_FILENAME}.sha256" \
    && tar -xzvf "${NUT_FILENAME}" \
    && cd "${NUT_FILENAME%.tar.gz}" \
    && ./configure --prefix=/usr --datadir=/usr/share/nut --sysconfdir=/etc/nut --with-drvpath=/usr/libexec/nut \
        --with-usb --with-hidapi --with-modbus --with-snmp --with-ssl \
    && make \
    && make DESTDIR=/app install

FROM base AS app

# Copy root filesystem
COPY rootfs /

# Copy nut files
COPY --from=builder /app /

# Setup app
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libhidapi-hidraw0=0.14.0-1+b2 \
        libmodbus5=3.1.11-2 \
        libsnmp40t64=5.9.4+dfsg-2+deb13u1 \
        libusb-1.0-0=2:1.0.28-1 \
        libxml2=2.12.7+dfsg+really2.9.14-2.1+deb13u2 \
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
