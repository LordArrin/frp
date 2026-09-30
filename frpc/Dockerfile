# syntax=docker/dockerfile:1

ARG FRP_VERSION=v0.71.0

FROM --platform=$BUILDPLATFORM alpine:latest AS source
ARG FRP_VERSION
RUN apk upgrade --no-cache && \
    apk add --no-cache curl tar && \
    mkdir -p /src && \
    curl -sSL "https://github.com/fatedier/frp/archive/refs/tags/${FRP_VERSION}.tar.gz" | \
    tar -xz --strip-components=1 -C /src

FROM --platform=$BUILDPLATFORM node:alpine AS frontend-builder
WORKDIR /src
COPY --from=source /src /src
WORKDIR /src/web/frpc
RUN apk upgrade --no-cache && \
    node -e 'const p = require("./package.json"); p.allowScripts = p.allowScripts || {}; p.allowScripts.esbuild = true; p.allowScripts["@parcel/watcher"] = true; require("fs").writeFileSync("./package.json", JSON.stringify(p, null, 2));' && \
    npm install && \
    npm run build-only

FROM --platform=$BUILDPLATFORM golang:alpine AS backend-builder
ARG FRP_VERSION
ARG TARGETOS
ARG TARGETARCH
RUN apk upgrade --no-cache && apk add --no-cache make ca-certificates tzdata
WORKDIR /src
COPY --from=source /src /src
COPY --from=frontend-builder /src/web/frpc/dist ./web/frpc/dist
RUN set -eux; \
    ARCH="${TARGETARCH:-$(go env GOARCH)}"; \
    OS="${TARGETOS:-$(go env GOOS)}"; \
    echo ">>> Compiling frpc for OS=$OS ARCH=$ARCH (Version: ${FRP_VERSION})"; \
    if [ "$ARCH" = "amd64" ]; then \
        export GOAMD64="v3"; \
        echo ">>> Applied AVX2 optimizations (GOAMD64=v3)"; \
    fi; \
    CGO_ENABLED=0 GOOS=$OS GOARCH=$ARCH go build \
    -trimpath \
    -ldflags="-s -w -linkmode=internal" \
    -o /out/frpc ./cmd/frpc

FROM alpine:latest
RUN apk upgrade --no-cache && \
    apk add --no-cache ca-certificates tzdata \
    && addgroup -S frp \
    && adduser -S -G frp -h /nonexistent -s /sbin/nologin frp \
    && mkdir -p /tmp/frp \
    && chown -R frp:frp /tmp/frp
COPY --from=backend-builder /out/frpc /usr/local/bin/frpc

USER frp
WORKDIR /tmp/frp

EXPOSE 7000 7400

ENTRYPOINT ["frpc"]
CMD ["-c", "/tmp/frp/frpc.toml"]