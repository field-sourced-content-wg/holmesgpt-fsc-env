# Web Terminal tooling base (arbitrary-UID + interactive shell) + Holmes CLI.
#
# ubi8/python-311 alone breaks the console session: OpenShift assigns a random
# UID whose passwd shell is /sbin/nologin → "The terminal connection has closed."
# Official web-terminal-tooling images are built for that UID model.
#
# Tooling's microdnf only has Python 3.6; holmesgpt needs >=3.10, so install
# CPython via uv and expose `holmes` on PATH.
# Pin to digest used by Web Terminal Operator DWT on this demo line (rhel9).
# Tag "1.8" exists for rhel8 tooling, not rhel9.
ARG BASE_IMAGE=registry.redhat.io/web-terminal/web-terminal-tooling-rhel9@sha256:f985f3d5a9940b3cf5e231d35973d5295013242d0220756241ed5c2be0744537
FROM ${BASE_IMAGE}

USER 0

ARG UV_VERSION=0.6.14
ARG HOLMESGPT_VERSION=0.43.0

ENV UV_TOOL_BIN_DIR=/usr/local/bin \
    UV_TOOL_DIR=/opt/uv-tools \
    UV_PYTHON_INSTALL_DIR=/opt/uv-python \
    PATH="/usr/local/bin:${PATH}"

RUN set -eux; \
    curl -fsSL -o /tmp/uv.tar.gz \
      "https://github.com/astral-sh/uv/releases/download/${UV_VERSION}/uv-x86_64-unknown-linux-gnu.tar.gz"; \
    tar -xzf /tmp/uv.tar.gz -C /usr/local/bin --strip-components=1; \
    rm -f /tmp/uv.tar.gz; \
    uv python install 3.11; \
    uv tool install "holmesgpt==${HOLMESGPT_VERSION}" --python 3.11; \
    holmes version; \
    chmod -R g+rwX /opt/uv-tools /opt/uv-python /usr/local/bin/holmes || true

COPY scripts/holmes-demo-env /usr/local/bin/holmes-demo-env
RUN chmod 0755 /usr/local/bin/holmes-demo-env

# Same non-root convention as the stock tooling image.
USER 1001
