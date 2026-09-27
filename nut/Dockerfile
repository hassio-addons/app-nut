ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:9.5.0
# hadolint ignore=DL3006
FROM ${BUILD_FROM} AS base

FROM base AS builder

ARG NUT_VERSION=2.8.5
ARG LIBUSB_VERSION=1.0.28
ARG LIBUSB_SHA256=966bb0d231f94a474eaae2e67da5ec844d3527a1f386456394ff432580634b29

# Patches applied to the NUT source before building
COPY patches /patches

# Setup builder
RUN \
    apt-get update \
    && apt-get install -y --no-install-recommends \
        autoconf=2.72-3.1 \
        automake=1:1.17-4 \
        build-essential=12.12 \
        libltdl-dev=2.5.4-4 \
        libmodbus-dev=3.1.11-2 \
        libsnmp-dev=5.9.4+dfsg-2+deb13u1 \
        libssl-dev=3.5.7-1~deb13u2 \
        libxml2-dev=2.12.7+dfsg+really2.9.14-2.1+deb13u3 \
        libtool=2.5.4-4 \
        patch=2.8-2 \
        pkg-config=1.8.1-4 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Use kernel hotplug events: host udev broadcasts do not reach bridge networking.
# Keep this in /usr/local so distro dependencies cannot overwrite it.
# Run upstream tests except stress_mt, which requires attached USB devices (#1373).
# The netlink backend needs a nonempty usbfs root; provide an empty bus for CI.
# hadolint ignore=DL3003
RUN \
    mkdir -p /app /build/libusb \
    && cd /build/libusb \
    && LIBUSB_FILENAME="libusb-${LIBUSB_VERSION}.tar.bz2" \
    && curl -fL -O "https://github.com/libusb/libusb/releases/download/v${LIBUSB_VERSION}/${LIBUSB_FILENAME}" \
    && test "$(sha256sum "${LIBUSB_FILENAME}")" = "${LIBUSB_SHA256}  ${LIBUSB_FILENAME}" \
    && tar -xjvf "${LIBUSB_FILENAME}" \
    && cd "${LIBUSB_FILENAME%.tar.bz2}" \
    && ./configure --prefix=/usr/local --libdir=/usr/local/lib \
        --disable-udev --disable-static --enable-tests-build \
    && make \
    && mkdir -p /dev/bus/usb/001 \
    && make check TESTS="stress set_option init_context" VERBOSE=1 \
    && rmdir /dev/bus/usb/001 \
    && make install \
    && make DESTDIR=/app install \
    && ldconfig

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
    && for nut_patch in /patches/*.patch; do \
        echo "Applying ${nut_patch}..." \
        && patch -p1 --batch --forward -i "${nut_patch}"; \
    done \
    && PKG_CONFIG_PATH=/usr/local/lib/pkgconfig \
        LDFLAGS="-Wl,-rpath,/usr/local/lib" \
        ./configure --prefix=/usr --datadir=/usr/share/nut --sysconfdir=/etc/nut --with-drvpath=/usr/libexec/nut \
        --with-usb --with-modbus --with-snmp --with-ssl \
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
        libltdl7=2.5.4-4 \
        libmodbus5=3.1.11-2 \
        libsnmp40t64=5.9.4+dfsg-2+deb13u1 \
        libusb-1.0-0=2:1.0.28-1 \
        libxml2=2.12.7+dfsg+really2.9.14-2.1+deb13u3 \
        usbutils=1:018-2 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Verify NUT's runpath wins over usbutils' Debian dependency on both architectures.
RUN \
    ldconfig \
    && case "$(ldd /usr/libexec/nut/usbhid-ups)" in \
        *"libusb-1.0.so.0 => /usr/local/lib/libusb-1.0.so.0 "*) ;; \
        *) echo "NUT is not loading the kernel-backend libusb"; exit 1 ;; \
    esac \
    && LIBUSB_DEPENDENCIES="$(ldd /usr/local/lib/libusb-1.0.so.0)" \
    && case "${LIBUSB_DEPENDENCIES}" in \
        *libudev*) echo "The kernel-backend libusb must not depend on libudev"; exit 1 ;; \
    esac

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
