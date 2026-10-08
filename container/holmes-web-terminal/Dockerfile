# Web Terminal image with working Holmes CLI.
#
# RHEL8 tooling only provides Python 3.6 via microdnf, but holmesgpt needs
# Python >=3.10. The Holmes server image already has a working venv + CLI, so
# use it as the base and copy OpenShift CLIs from the Web Terminal tooling image.
ARG HOLMES_IMAGE=docker.io/robustadev/holmes:0.43.0
ARG TOOLING_IMAGE=registry.redhat.io/web-terminal/web-terminal-tooling-rhel8:1.8

FROM ${TOOLING_IMAGE} AS tooling

FROM ${HOLMES_IMAGE}

USER 0

# OpenShift / Kubernetes CLIs (tooling:1.8 keeps them under /usr/bin).
# Install into /usr/local/bin so they win over any empty stubs on PATH.
COPY --from=tooling /usr/bin/oc /usr/local/bin/oc
COPY --from=tooling /usr/bin/kubectl /usr/local/bin/kubectl
COPY --from=tooling /usr/bin/helm /usr/local/bin/helm
COPY --from=tooling /usr/local/bin/wtoctl /usr/local/bin/wtoctl
COPY --from=tooling /usr/local/bin/wtoctl_help.sh /usr/local/bin/wtoctl_help.sh
COPY --from=tooling /usr/local/bin/wtoctl_jq.sh /usr/local/bin/wtoctl_jq.sh

COPY scripts/holmes /usr/local/bin/holmes
COPY scripts/holmes-demo-env /usr/local/bin/holmes-demo-env
RUN chmod 0755 \
      /usr/local/bin/holmes \
      /usr/local/bin/holmes-demo-env \
      /usr/local/bin/oc \
      /usr/local/bin/kubectl \
      /usr/local/bin/helm \
    && /usr/local/bin/holmes version \
    && /usr/local/bin/oc version --client
# Web Terminal tooling runs as non-root and stays up with a no-op command.
USER 1001
CMD ["tail", "-f", "/dev/null"]
