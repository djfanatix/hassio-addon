# Fetch the requested evcc branch as part of the Home Assistant add-on build.
FROM --platform=$BUILDPLATFORM alpine:3.23 AS source

ARG EVCC_REF=test/optimizer-automatic
ARG EVCC_REPO=https://github.com/djfanatix/evcc.git

# Bust the build cache whenever EVCC_REF moves: this response body changes with
# every new commit, so the clone below is re-run instead of served from cache.
ADD https://api.github.com/repos/djfanatix/evcc/commits/${EVCC_REF} /tmp/evcc-head.json

RUN apk add --no-cache git \
    && git clone --depth 1 --branch "${EVCC_REF}" "${EVCC_REPO}" /src

# Build the frontend with the toolchain used by the selected evcc branch.
FROM --platform=$BUILDPLATFORM ghcr.io/voidzero-dev/vite-plus:0.2.6 AS frontend

USER root
WORKDIR /build
COPY --from=source /src/.node-version /src/package*.json ./
RUN npm ci

COPY --from=source /src/Makefile ./
COPY --from=source /src/*.js /src/*.ts /src/*.mts ./
COPY --from=source /src/.browserslistrc ./
COPY --from=source /src/assets ./assets
COPY --from=source /src/i18n ./i18n
RUN make ui

# Build the evcc executable for the target Home Assistant architecture.
FROM --platform=$BUILDPLATFORM golang:1.27.0-alpine AS builder

RUN apk add --no-cache git make patch tzdata ca-certificates \
    && update-ca-certificates

ARG TARGETOS
ARG TARGETARCH
ARG TARGETVARIANT
ARG GOARM=${TARGETVARIANT#v}
ARG RELEASE=0

WORKDIR /build
ENV GOCACHE=/root/.cache/go-build
ENV GOMODCACHE=/root/.cache/go-mod

COPY --from=source /src/ ./
RUN go mod download
RUN make install
RUN make patch-asn1
RUN make assets
COPY --from=frontend /build/dist ./dist

RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} GOARM=${GOARM} \
    RELEASE=${RELEASE} make build

# Keep the runtime image small while retaining the upstream add-on entrypoint.
FROM alpine:3.23

WORKDIR /app
ENV TZ=Europe/Berlin

COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /build/evcc /usr/local/bin/evcc
COPY --from=source /src/packaging/docker/bin/entrypoint.sh /app/entrypoint.sh

EXPOSE 4712/tcp 5353/udp 7070/tcp 7090/udp 8887/tcp 8899/udp 9522/udp

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD wget -qO /dev/null http://localhost:7070 || exit 1

ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["evcc"]