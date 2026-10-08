# Holmes CLI on top of OpenShift Web Terminal tooling (oc, kubectl, etc.).
# Built in-cluster via BuildConfig → ImageStream; selected as Web Terminal custom image.
#
# Base: official Web Terminal tooling (KCS 6962984 / 7055855 patterns).
ARG BASE_IMAGE=registry.redhat.io/web-terminal/web-terminal-tooling-rhel8:1.8
FROM ${BASE_IMAGE}

USER 0

# Prefer pip if present on the tooling image; otherwise install python3-pip.
RUN set -eux; \
    if command -v pip3 >/dev/null 2>&1; then \
      pip3 install --no-cache-dir 'holmesgpt'; \
    elif command -v microdnf >/dev/null 2>&1; then \
      microdnf -y install python3 python3-pip && microdnf clean all && \
      pip3 install --no-cache-dir 'holmesgpt'; \
    else \
      dnf -y install python3 python3-pip && dnf clean all && \
      pip3 install --no-cache-dir 'holmesgpt'; \
    fi; \
    holmes --help >/dev/null

COPY scripts/holmes-demo-env /usr/local/bin/holmes-demo-env
RUN chmod 0755 /usr/local/bin/holmes-demo-env

# Web Terminal tooling runs as non-root (UID 1001).
USER 1001
