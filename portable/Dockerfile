# Builds a fully static, portable https_dns_proxy binary with HTTP/3 (QUIC) support.
# Dependencies are built from source (libev and c-ares come from Alpine), based on upstream development_build_with_http3.sh.
#
# Why some libraries are built from source instead of taken from Alpine packages
# (the goal is one statically linked executable, so only static libraries (.a) can be used):
#   curl    - Alpine's libcurl is built without HTTP/3 (no ngtcp2/nghttp3), and its static
#             library also pulls in brotli, zstd, zlib, libidn2 and libpsl, which a DoH
#             proxy does not use. A custom build keeps only what is needed.
#   ngtcp2  - QUIC for curl. Its OpenSSL crypto backend must be built against the same
#             OpenSSL that is linked into the binary.
#   nghttp3 - HTTP/3 for curl, built together with ngtcp2 to keep matching versions.
#   OpenSSL - Alpine ships the 3.5 LTS line, which would also work (ngtcp2 needs >= 3.5);
#             it is built from source to get the latest release.
#   nghttp2 - HTTP/2 for curl; Alpine has a static package, built from source to stay on
#             the latest release together with the rest of the HTTP stack.
# libev and c-ares are taken from Alpine: their packages already include static
# libraries and match the latest upstream releases.
#
# Run from the repository root.
#
# Portable binary only:
#   docker buildx build -f portable/Dockerfile --target binary --output type=local,dest=./out .
#   -> ./out/https_dns_proxy
#
# Docker image (single layer on top of an empty image):
#   docker buildx build -f portable/Dockerfile --target image -t https_dns_proxy .
FROM alpine:3.24 AS build

ARG OPENSSL_VERSION=openssl-3.6.5
ARG NGHTTP3_VERSION=v1.18.0
ARG NGTCP2_VERSION=v1.25.0
ARG NGHTTP2_VERSION=v1.70.0
ARG CURL_VERSION=curl-8_22_0

RUN apk add --no-cache build-base linux-headers cmake git perl pkgconf autoconf automake libtool curl \
    libev-dev c-ares-dev ca-certificates

ENV PREFIX=/opt/static
ENV PKG_CONFIG_PATH=/opt/static/lib/pkgconfig
ENV CFLAGS="-O2 -fPIC"
WORKDIR /usr/src

RUN git clone --depth 1 -b ${OPENSSL_VERSION} https://github.com/openssl/openssl \
 && cd openssl \
 && ./config --prefix=$PREFIX --libdir=lib --openssldir=/etc/ssl no-shared no-docs no-tests \
 && make -j$(nproc) && make install_dev

RUN git clone --depth 1 -b ${NGHTTP3_VERSION} https://github.com/ngtcp2/nghttp3 \
 && cd nghttp3 && git submodule update --init --depth 1 \
 && autoreconf -fi \
 && ./configure --prefix=$PREFIX --enable-lib-only --disable-shared --enable-static \
 && make -j$(nproc) && make install

RUN git clone --depth 1 -b ${NGTCP2_VERSION} https://github.com/ngtcp2/ngtcp2 \
 && cd ngtcp2 && git submodule update --init --depth 1 \
 && autoreconf -fi \
 && ./configure --prefix=$PREFIX --enable-lib-only --disable-shared --enable-static --with-openssl \
 && make -j$(nproc) && make install

RUN git clone --depth 1 -b ${NGHTTP2_VERSION} https://github.com/nghttp2/nghttp2 \
 && cd nghttp2 && git submodule update --init --depth 1 \
 && autoreconf -fi \
 && ./configure --prefix=$PREFIX --enable-lib-only --disable-shared --enable-static \
 && make -j$(nproc) && make install

RUN git clone --depth 1 -b ${CURL_VERSION} https://github.com/curl/curl \
 && cd curl \
 && autoreconf -fi \
 && ./configure --prefix=$PREFIX --disable-shared --enable-static \
      --with-openssl=$PREFIX --with-nghttp2=$PREFIX --with-ngtcp2=$PREFIX --with-nghttp3=$PREFIX \
      --enable-ares \
      --with-ca-bundle=/etc/ssl/certs/ca-certificates.crt --with-ca-path=/etc/ssl/certs \
      --without-libpsl --without-libidn2 --without-brotli --without-zstd --without-zlib \
      --without-libssh2 --without-gssapi --without-librtmp \
      --disable-ldap --disable-ldaps --disable-docs --disable-manual \
 && make -j$(nproc) && make install

COPY . /usr/src/https_dns_proxy
WORKDIR /usr/src/https_dns_proxy

# libsystemd has no static library, so the build goes without sd_notify support
RUN cmake . -DUSE_CLANG_TIDY=OFF \
      -DCMAKE_PREFIX_PATH=$PREFIX \
      -DCUSTOM_LIBCURL_INSTALL_PATH=$PREFIX \
      -DCMAKE_EXE_LINKER_FLAGS="-static" \
      -DCMAKE_C_STANDARD_LIBRARIES="$(pkg-config --static --libs libcurl libcares)" \
 && make -j$(nproc) \
 && strip https_dns_proxy

# Everything the image needs is gathered in one directory, so it is copied as a single layer.
# /etc/services is needed by c-ares to resolve the "https" service name when bootstrapping
# a resolver given by hostname.
RUN mkdir -p /rootfs/etc/ssl/certs \
 && cp https_dns_proxy /rootfs/https_dns_proxy \
 && cp /etc/ssl/certs/ca-certificates.crt /rootfs/etc/ssl/certs/ \
 && cp /etc/services /rootfs/etc/services

FROM scratch AS binary
COPY --from=build /usr/src/https_dns_proxy/https_dns_proxy /https_dns_proxy

FROM scratch AS image
COPY --from=build /rootfs/ /
USER 65534:65534
EXPOSE 5053/udp 5053/tcp
# Listen on all interfaces inside the container; a later -a overrides it
ENTRYPOINT ["/https_dns_proxy", "-a", "0.0.0.0"]
