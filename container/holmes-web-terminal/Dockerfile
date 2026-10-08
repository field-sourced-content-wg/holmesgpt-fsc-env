# Web Terminal tooling (RHEL8/glibc, includes oc) + Holmes CLI via Python 3.11.
#
# Cannot copy CLI from robustadev/holmes: that image is Alpine/musl, while
# tooling's oc is a glibc binary ("No such file or directory" if mixed).
ARG BASE_IMAGE=registry.redhat.io/web-terminal/web-terminal-tooling-rhel8:1.8
FROM ${BASE_IMAGE}

USER 0

# holmesgpt on PyPI requires Python >=3.10; tooling's default platform python is 3.6.
RUN set -eux; \
    microdnf -y module enable python311; \
    microdnf -y install python3.11 python3.11-pip python3.11-setuptools; \
    microdnf clean all; \
    python3.11 -m pip install --no-cache-dir --upgrade pip setuptools wheel; \
    python3.11 -m pip install --no-cache-dir 'holmesgpt==0.43.0'; \
    ln -sf "$(python3.11 -c 'import shutil; print(shutil.which("holmes"))')" /usr/local/bin/holmes; \
    holmes version

COPY scripts/holmes-demo-env /usr/local/bin/holmes-demo-env
RUN chmod 0755 /usr/local/bin/holmes-demo-env

USER 1001
