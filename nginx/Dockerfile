# syntax=docker/dockerfile:1.7

ARG ALPINE_VERSION=3.22

FROM alpine:${ALPINE_VERSION} AS builder

ARG NGINX_VERSION=1.30.3
ARG NGINX_SHA256=e5823dc6f45610993def93ebf6cfce68264af4958c77e874b7d20f3709001b8f
ARG HEADERS_MORE_VERSION=0.40
ARG HEADERS_MORE_SHA256=c14cb5e6c998590c209efbf77bd7637ce2cab02332e4192756a8af5b26ba4284
ARG NGX_BROTLI_COMMIT=a71f9312c2deb28875acc7bacfdd5695a111aa53
ARG MAKE_JOBS=2

WORKDIR /usr/local/src

RUN apk add --no-cache \
        build-base \
        cmake \
        curl \
        git \
        linux-headers \
        openssl-dev \
        pcre2-dev \
        zlib-dev

RUN set -eux; \
    curl -fsSLo nginx.tar.gz \
        "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz"; \
    echo "${NGINX_SHA256}  nginx.tar.gz" | sha256sum -c -; \
    mkdir nginx; \
    tar -xzf nginx.tar.gz --strip-components=1 -C nginx; \
    curl -fsSLo headers-more.tar.gz \
        "https://github.com/openresty/headers-more-nginx-module/archive/refs/tags/v${HEADERS_MORE_VERSION}.tar.gz"; \
    echo "${HEADERS_MORE_SHA256}  headers-more.tar.gz" | sha256sum -c -; \
    mkdir headers-more; \
    tar -xzf headers-more.tar.gz --strip-components=1 -C headers-more; \
    git clone https://github.com/google/ngx_brotli.git ngx_brotli; \
    cd ngx_brotli; \
    git checkout --detach "${NGX_BROTLI_COMMIT}"; \
    git submodule update --init --recursive --depth 1

RUN set -eux; \
    cmake \
        -S /usr/local/src/ngx_brotli/deps/brotli \
        -B /usr/local/src/ngx_brotli/deps/brotli/out \
        -DCMAKE_BUILD_TYPE=Release \
        -DBUILD_SHARED_LIBS=OFF \
        -DBROTLI_DISABLE_TESTS=ON; \
    cmake \
        --build /usr/local/src/ngx_brotli/deps/brotli/out \
        --config Release \
        --target brotlienc \
        --parallel "${MAKE_JOBS}"; \
    cd /usr/local/src/nginx; \
    ./configure \
        --prefix=/etc/nginx \
        --with-http_gzip_static_module \
        --with-http_realip_module \
        --with-http_ssl_module \
        --with-http_stub_status_module \
        --with-http_sub_module \
        --with-http_v2_module \
        --with-http_v3_module \
        --with-stream \
        --with-stream_ssl_module \
        --with-stream_ssl_preread_module \
        --add-module=/usr/local/src/ngx_brotli \
        --add-module=/usr/local/src/headers-more; \
    make -j"${MAKE_JOBS}"; \
    make install

FROM alpine:${ALPINE_VERSION}

LABEL org.opencontainers.image.authors="bywebol <629730936@qq.com>" \
      org.opencontainers.image.description="NGINX with HTTP/2, HTTP/3, Brotli and headers-more modules"

ENV TZ=Asia/Shanghai \
    PATH=/etc/nginx/sbin:$PATH

RUN apk add --no-cache \
        libcrypto3 \
        libssl3 \
        pcre2 \
        tzdata \
        zlib \
    && ln -snf "/usr/share/zoneinfo/${TZ}" /etc/localtime \
    && echo "${TZ}" > /etc/timezone

COPY --from=builder /etc/nginx /etc/nginx

RUN ln -sf /dev/stdout /etc/nginx/logs/access.log \
    && ln -sf /dev/stderr /etc/nginx/logs/error.log

EXPOSE 80 443 443/udp

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]
