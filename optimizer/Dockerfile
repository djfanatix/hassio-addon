# Fetch the requested optimizer branch as part of the Home Assistant add-on build.
FROM --platform=$BUILDPLATFORM alpine:3.23 AS source

ARG OPTIMIZER_REF=feat/per-battery-efficiency-v2
ARG OPTIMIZER_REPO=https://github.com/djfanatix/optimizer.git

# Bust the build cache whenever OPTIMIZER_REF moves: this response body changes with
# every new commit, so the clone below is re-run instead of served from cache.
ADD https://api.github.com/repos/djfanatix/optimizer/commits/${OPTIMIZER_REF} /tmp/optimizer-head.json

RUN apk add --no-cache git \
    && git clone --depth 1 --branch "${OPTIMIZER_REF}" "${OPTIMIZER_REPO}" /src

# Build the Python venv, mirroring the optimizer repo's own Dockerfile (uv-managed deps).
FROM python:3.13-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

COPY --from=source /src/uv.lock /src/pyproject.toml ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-install-project --no-editable --no-group dev

COPY --from=source /src /app

RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-editable --no-group dev

ADD --checksum=sha256:f551e7b843e25becee466a447118f6f44f219c4e46cfb4670829ecd3cf47e7d8 \
    https://github.com/coin-or/Cbc/releases/download/releases%2F2.10.10/Cbc-releases.2.10.10-x86_64-ubuntu22-gcc1130-static.tar.gz \
    /tmp/cbc.tar.gz

# Runtime image: only the venv, no source tree or build toolchain.
FROM python:3.13-slim

RUN groupadd -r app && useradd -r -g app -s /bin/false app

COPY --from=builder --chown=app:app /app/.venv /app/.venv

ARG TARGETARCH

# pulp bundles CBC 2.10.10 for arm64 but 2.10.3 from 2019 for amd64, and resolves the solver by a
# fixed path with no override, so the bundled amd64 binary is linked to the fetched 2.10.10 build.
RUN --mount=from=builder,source=/tmp/cbc.tar.gz,target=/tmp/cbc.tar.gz set -eu; \
    if [ "$TARGETARCH" = "amd64" ]; then \
        tar -xzf /tmp/cbc.tar.gz -C /usr/local ./bin/cbc; \
        find /app/.venv -path '*/pulp/solverdir/cbc/*/cbc' -exec ln -sf /usr/local/bin/cbc {} \; ; \
    fi; \
    echo | "$(/app/.venv/bin/python -c 'from pulp.apis.coin_api import pulp_cbc_path; print(pulp_cbc_path)')" \
        | grep -q 'Version: 2.10.10'

ENV PYTHONUNBUFFERED=1
ENV OPTIMIZER_TIME_LIMIT=30
ENV OPTIMIZER_NUM_THREADS=2
# the access log is the only source of per request latency, keep the format lean.
# --timeout must stay well above OPTIMIZER_TIME_LIMIT: gunicorn SIGKILLs a worker that
# exceeds it, and the solve() call alone can take the full OPTIMIZER_TIME_LIMIT before
# response building/serialization even starts. Gunicorn's own default is 30s, which
# collided with OPTIMIZER_TIME_LIMIT=30 and killed every request.
ENV GUNICORN_CMD_ARGS="--workers 4 --max-requests 32 --timeout 90 --config python:optimizer.gunicorn_conf --access-logfile - --access-logformat '%(m)s %(U)s %(s)s %(D)s'"

EXPOSE 7050/tcp

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD /app/.venv/bin/python -c "import urllib.request; urllib.request.urlopen('http://localhost:7050/optimize/health')" || exit 1

USER app
CMD ["/app/.venv/bin/gunicorn", "--bind", "0.0.0.0:7050", "optimizer.app:app"]
