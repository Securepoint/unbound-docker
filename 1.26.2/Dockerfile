FROM debian:trixie AS openssl
LABEL maintainer="Securepoint GmbH"

ENV VERSION_OPENSSL=openssl-4.0.3 \
    SHA256_OPENSSL=325b5c806167c13b40b1ffeadfe0248197c00eccc4cf123ec1e28d2d2fd216d9 \
    SOURCE_OPENSSL=https://github.com/openssl/openssl/releases/download/ \
    OPGP_OPENSSL_1=B146647E45A7B33947AB226B2A2C87D161692D40

WORKDIR /tmp/src

RUN set -e -x && \
    build_deps="build-essential ca-certificates curl dirmngr gnupg libidn2-0-dev libssl-dev" && \
    DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y --no-install-recommends \
    $build_deps && \
    curl -L "${SOURCE_OPENSSL}${VERSION_OPENSSL}/${VERSION_OPENSSL}.tar.gz" -o openssl.tar.gz && \
    echo "${SHA256_OPENSSL} ./openssl.tar.gz" | sha256sum -c - && \
    curl -L "${SOURCE_OPENSSL}${VERSION_OPENSSL}/${VERSION_OPENSSL}.tar.gz.asc" -o openssl.tar.gz.asc && \
    GNUPGHOME="$(mktemp -d)" && \
    export GNUPGHOME && \
    gpg --no-tty --keyserver keyserver.ubuntu.com --recv-keys "$OPGP_OPENSSL_1" && \
    gpg --batch --verify openssl.tar.gz.asc openssl.tar.gz && \
    tar xzf openssl.tar.gz && \
    cd $VERSION_OPENSSL && \
    ./config \
    --prefix=/opt/openssl \
    --openssldir=/opt/openssl \
    no-weak-ssl-ciphers \
    no-ssl3 \
    no-shared \
    enable-ec_nistp_64_gcc_128 \
    -DOPENSSL_NO_HEARTBEATS \
    -fstack-protector-strong && \
    make depend && \
    nproc | xargs -I % make -j% && \
    make install_sw && \
    apt-get purge -y --auto-remove \
    $build_deps && \
    rm -rf \
    /tmp/* \
    /var/tmp/* \
    /var/lib/apt/lists/*

FROM debian:trixie AS unbound
LABEL maintainer="Securepoint GmbH"

ENV NAME=unbound \
    UNBOUND_VERSION=1.26.2 \
    UNBOUND_SHA256=d8ee7aa6bbb3ff1be7d9e57637d0d0ec2f18273a8efff1a0afe2c665858b96f7 \
    UNBOUND_DOWNLOAD_URL=https://nlnetlabs.nl/downloads/unbound/unbound-1.26.2.tar.gz

WORKDIR /tmp/src

COPY --from=openssl /opt/openssl /opt/openssl

RUN build_deps="curl gcc libc-dev libevent-dev libexpat1-dev libnghttp2-dev make" && \
    set -x && \
    DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y --no-install-recommends \
    $build_deps \
    bsdmainutils \
    ca-certificates \
    ldnsutils \
    libevent-2.1-7 \
    libhiredis-dev \
    libexpat1 \
    libprotobuf-c-dev \
    protobuf-c-compiler && \
    curl -sSL $UNBOUND_DOWNLOAD_URL -o unbound.tar.gz && \
    echo "${UNBOUND_SHA256} *unbound.tar.gz" | sha256sum -c - && \
    tar xzf unbound.tar.gz && \
    rm -f unbound.tar.gz && \
    cd unbound-1.26.2 && \
    groupadd _unbound && \
    useradd -g _unbound -s /dev/null -d /etc _unbound && \
    ./configure \
    --disable-dependency-tracking \
    --prefix=/opt/unbound \
    --with-pthreads \
    --with-username=_unbound \
    --with-ssl=/opt/openssl \
    --with-libevent \
    --with-libhiredis \
    --with-libnghttp2 \
    --enable-dnstap \
    --enable-tfo-server \
    --enable-cachedb \
    --enable-tfo-client \
    --enable-event-api \
    --enable-subnet && \
    make install && \
    mv /opt/unbound/etc/unbound/unbound.conf /opt/unbound/etc/unbound/unbound.conf.example && \
    apt-get purge -y --auto-remove \
    $build_deps && \
    rm -rf \
    /opt/unbound/share/man \
    /tmp/* \
    /var/tmp/* \
    /var/lib/apt/lists/*


FROM debian:trixie
LABEL maintainer="Securepoint GmbH"

ENV NAME=unbound \
    SUMMARY="${NAME} is a validating, recursive, and caching DNS resolver." \
    DESCRIPTION="${NAME} is a validating, recursive, and caching DNS resolver."

WORKDIR /tmp/src

COPY --from=unbound /opt /opt

RUN set -x && \
    DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y --no-install-recommends \
    bsdmainutils \
    ca-certificates \
    ldnsutils \
    libevent-2.1-7 \
    libhiredis1.1.0 \
    libnghttp2-14 \
    libexpat1 \
    libprotobuf-c1 && \
    groupadd _unbound && \
    useradd -g _unbound -s /dev/null -d /etc _unbound && \
    apt-get purge -y --auto-remove \
    $build_deps && \
    rm -rf \
    /opt/unbound/share/man \
    /tmp/* \
    /var/tmp/* \
    /var/lib/apt/lists/*

COPY data/ /

RUN chmod +x /unbound.sh

WORKDIR /opt/unbound/

ENV PATH /opt/unbound/sbin:"$PATH"

LABEL org.opencontainers.image.version=${UNBOUND_VERSION} \
    org.opencontainers.image.title="securepoint/unbound-docker" \
    org.opencontainers.image.description="a validating, recursive, and caching DNS resolver" \
    org.opencontainers.image.url="https://github.com/Securepoint/unbound-docker" \
    org.opencontainers.image.vendor="Securepoint GmbH" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.source="https://github.com/Securepoint/unbound-docker"

EXPOSE 53/tcp
EXPOSE 53/udp

HEALTHCHECK --interval=30s --timeout=30s --start-period=10s --retries=3 CMD drill @127.0.0.1 cloudflare.com || exit 1

CMD ["/unbound.sh"]
