# Holmes CLI on OpenShift Web Terminal tooling.
# PyPI holmesgpt needs Python >=3.10; the RHEL8 tooling image only has 3.6 via
# microdnf, so we copy the venv + CLI from the official Holmes image instead.
ARG HOLMES_IMAGE=docker.io/robustadev/holmes:0.43.0
ARG BASE_IMAGE=registry.redhat.io/web-terminal/web-terminal-tooling-rhel8:1.8

FROM ${HOLMES_IMAGE} AS holmes

FROM ${BASE_IMAGE}

USER 0

COPY --from=holmes /venv /opt/holmes/venv
COPY --from=holmes /app /opt/holmes/app

COPY scripts/holmes /usr/local/bin/holmes
COPY scripts/holmes-demo-env /usr/local/bin/holmes-demo-env
RUN chmod 0755 /usr/local/bin/holmes /usr/local/bin/holmes-demo-env \
    && /usr/local/bin/holmes version

# Web Terminal tooling runs as non-root (UID 1001).
USER 1001
