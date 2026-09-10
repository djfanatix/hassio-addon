# Fetch the requested evcc branch as part of the Home Assistant add-on build.
FROM alpine:3.23 AS source

ARG EVCC_REF=test/optimizer

RUN apk add --no-cache git \
    && git clone --depth 1 --branch "${EVCC_REF}" https://github.com/evcc-io/evcc.git /src

# Build the frontend with the toolchain used by the selected evcc branch.
FROM --platform=$BUILDPLATFORM ghcr.io/voidzero-dev/vite-plus:0.2.6 AS frontend

USER root
WORKDIR /build
COPY --from=source /src/ ./

RUN npm ci \
    && make ui

# Build the evcc executable for the target Home Assistant architecture.
FROM --platform=$BUILDPLATFORM golang:1.27.0-alpine AS builder

RUN apk add --no-cache git make patch tzdata ca-certificates \
    && update-ca-certificates

ARG TARGETOS
ARG TARGETARCH
ARG TARGETVARIANT
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

RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} GOARM=${TARGETVARIANT#v} \
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