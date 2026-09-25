# syntax=docker/dockerfile:1
# From-source, multi-arch rebuild of the MinIO image we used to pull as
#   minio/minio@sha256:14cea493d9a34af32f524e538b8346cf79f3321eff8e708c1e2960462bd8936e
# which was the linux/amd64 manifest of RELEASE.2025-09-07T16-13-09Z. MinIO
# pulled their images from Docker Hub and quay, and dl.min.io (which upstream's
# Dockerfile.release downloads prebuilt binaries from) now returns 410, so we
# build the binaries ourselves from the archived repos.
#
# Mirrors upstream Dockerfile.release at that tag:
#   - minio: `make build` flags (CGO off, -tags kqueue, -trimpath, gen-ldflags)
#   - mc: upstream shipped whatever was latest at build time; we pin the last
#     mc release before the server release
#   - curl: static curl 8.11.0, same version the old image carried
#   - runtime: ubi9/ubi-micro 9.6 + upstream docker-entrypoint.sh

ARG MINIO_TAG=RELEASE.2025-09-07T16-13-09Z
ARG MC_TAG=RELEASE.2025-08-13T08-35-41Z
ARG CURL_VERSION=v8.11.0

FROM --platform=$BUILDPLATFORM alpine:3.22 AS source
ARG MINIO_TAG
ARG MC_TAG
RUN apk add --no-cache git
RUN git clone --depth 1 --branch ${MINIO_TAG} https://github.com/minio/minio.git /src/minio
RUN git clone --depth 1 --branch ${MC_TAG} https://github.com/minio/mc.git /src/mc

# Cross-compile on the build platform so multi-arch builds don't run the Go
# toolchain under qemu.
FROM --platform=$BUILDPLATFORM golang:1.24.7-alpine AS builder
ARG TARGETOS
ARG TARGETARCH
ARG MINIO_TAG
ARG MC_TAG
# gen-ldflags.go shells out to `git log` for the commit id
RUN apk add --no-cache git
ENV CGO_ENABLED=0 GOTOOLCHAIN=local
COPY --from=source /src /src

# RELEASE.2025-09-07T16-13-09Z -> 2025-09-07T16:13:09Z, same sed as the
# upstream Makefile's hotfix-vars target.
WORKDIR /src/minio
RUN go mod download
RUN VERSION=$(echo ${MINIO_TAG} | sed -E 's/^RELEASE\.([0-9]+-[0-9]+-[0-9]+)T([0-9]+)-([0-9]+)-([0-9]+)Z$/\1T\2:\3:\4Z/') && \
    LDFLAGS=$(MINIO_RELEASE=RELEASE go run buildscripts/gen-ldflags.go "${VERSION}") && \
    GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -tags kqueue -trimpath -ldflags "${LDFLAGS}" -o /out/minio

WORKDIR /src/mc
RUN go mod download
RUN VERSION=$(echo ${MC_TAG} | sed -E 's/^RELEASE\.([0-9]+-[0-9]+-[0-9]+)T([0-9]+)-([0-9]+)-([0-9]+)Z$/\1T\2:\3:\4Z/') && \
    LDFLAGS=$(MC_RELEASE=RELEASE go run buildscripts/gen-ldflags.go "${VERSION}") && \
    GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -tags kqueue -trimpath -ldflags "${LDFLAGS}" -o /out/mc

FROM --platform=$BUILDPLATFORM alpine:3.22 AS curl
ARG TARGETARCH
ARG CURL_VERSION
RUN apk add --no-cache curl
# checksums from the release's sha256sum.txt
RUN case ${TARGETARCH} in \
      amd64) asset=curl-amd64;   sum=d18aa1f4e03b50b649491ca2c401cd8c5e89e72be91ff758952ad2ab5a83135d ;; \
      arm64) asset=curl-aarch64; sum=1b050abd1669f9a2ac29b34eb022cdeafb271dce5a4fb57d8ef8fadff6d7be1f ;; \
      *) echo "no static curl for ${TARGETARCH}" >&2; exit 1 ;; \
    esac && \
    mkdir /out && \
    curl -fsSL -o /out/curl https://github.com/moparisthebest/static-curl/releases/download/${CURL_VERSION}/${asset} && \
    echo "${sum}  /out/curl" | sha256sum -c - && \
    chmod +x /out/curl

FROM registry.access.redhat.com/ubi9/ubi-micro:9.6

ARG MINIO_TAG

LABEL name="MinIO" \
      vendor="MinIO Inc <dev@min.io>" \
      maintainer="MinIO Inc <dev@min.io>" \
      version="${MINIO_TAG}" \
      release="${MINIO_TAG}" \
      summary="MinIO is a High Performance Object Storage, API compatible with Amazon S3 cloud storage service." \
      org.opencontainers.image.source="https://github.com/novelai/public-images"

ENV MINIO_ACCESS_KEY_FILE=access_key \
    MINIO_SECRET_KEY_FILE=secret_key \
    MINIO_ROOT_USER_FILE=access_key \
    MINIO_ROOT_PASSWORD_FILE=secret_key \
    MINIO_KMS_SECRET_KEY_FILE=kms_master_key \
    MINIO_UPDATE_MINISIGN_PUBKEY="RWTx5Zr1tiHQLwG9keckT0c45M3AGeHD6IvimQHpyRywVWGbP1aVSGav" \
    MINIO_CONFIG_ENV_FILE=config.env \
    MC_CONFIG_DIR=/tmp/.mc

RUN chmod -R 777 /usr/bin

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /out/minio /out/mc /usr/bin/
COPY --from=curl /out/curl /usr/bin/

COPY --from=source /src/minio/CREDITS /licenses/CREDITS
COPY --from=source /src/minio/LICENSE /licenses/LICENSE
COPY --from=source /src/minio/dockerscripts/docker-entrypoint.sh /usr/bin/docker-entrypoint.sh

EXPOSE 9000
VOLUME ["/data"]

ENTRYPOINT ["/usr/bin/docker-entrypoint.sh"]
# add CMD so that the image works in GH CI, see our ci yaml for more details
CMD ["server", "/data", "--address=0.0.0.0:9000"]
