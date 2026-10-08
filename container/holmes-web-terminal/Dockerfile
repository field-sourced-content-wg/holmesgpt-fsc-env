# Custom Web Terminal image: ubi8/python-311 + holmes CLI + oc from Web Terminal tooling.
#
# - tooling has oc (RHEL8/glibc) but no Python >=3.10 module for pip holmesgpt
# - robustadev/holmes is Alpine/musl (cannot run RHEL oc)
# - mirror "stable" oc needs GLIBC_2.34+ (newer than UBI8)
# → combine ubi8/python-311 with oc/kubectl copied from tooling:1.8
ARG PYTHON_IMAGE=registry.access.redhat.com/ubi8/python-311:1
ARG TOOLING_IMAGE=registry.redhat.io/web-terminal/web-terminal-tooling-rhel8:1.8

FROM ${TOOLING_IMAGE} AS tooling

FROM ${PYTHON_IMAGE}

USER 0

COPY --from=tooling /usr/bin/oc /usr/local/bin/oc
COPY --from=tooling /usr/bin/kubectl /usr/local/bin/kubectl
COPY --from=tooling /usr/bin/helm /usr/local/bin/helm

RUN set -eux; \
    chmod 0755 /usr/local/bin/oc /usr/local/bin/kubectl /usr/local/bin/helm; \
    pip install --no-cache-dir 'holmesgpt==0.43.0'; \
    holmes version; \
    oc version --client

COPY scripts/holmes-demo-env /usr/local/bin/holmes-demo-env
RUN chmod 0755 /usr/local/bin/holmes-demo-env

USER 1001
CMD ["tail", "-f", "/dev/null"]
