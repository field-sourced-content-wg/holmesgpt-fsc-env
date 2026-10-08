# Custom Web Terminal image: Python 3.11 (glibc) + oc/kubectl + holmes CLI.
#
# Notes:
# - Web Terminal tooling (RHEL8) has oc but only Python 3.6 via microdnf (no python311 module).
# - robustadev/holmes is Alpine/musl — RHEL oc cannot run there.
# - ubi8/python-311 + official OpenShift client tarball is a compatible glibc stack.
ARG PYTHON_IMAGE=registry.access.redhat.com/ubi8/python-311:1
FROM ${PYTHON_IMAGE}

USER 0

ARG OC_CLIENT_URL=https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz

RUN set -eux; \
    dnf -y install tar gzip findutils; \
    curl -fsSL -o /tmp/openshift-client-linux.tar.gz "${OC_CLIENT_URL}"; \
    tar -xzf /tmp/openshift-client-linux.tar.gz -C /usr/local/bin oc kubectl; \
    rm -f /tmp/openshift-client-linux.tar.gz; \
    chmod 0755 /usr/local/bin/oc /usr/local/bin/kubectl; \
    pip install --no-cache-dir 'holmesgpt==0.43.0'; \
    holmes version; \
    oc version --client; \
    dnf clean all

COPY scripts/holmes-demo-env /usr/local/bin/holmes-demo-env
RUN chmod 0755 /usr/local/bin/holmes-demo-env

# Web Terminal expects a non-root user and a long-running process.
USER 1001
CMD ["tail", "-f", "/dev/null"]
