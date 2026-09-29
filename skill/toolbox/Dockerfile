# syntax=docker/dockerfile:1
# Linux toolbox for the impeccable-python skill: its command-line tools, pinned.
# Build and use it through scripts/impeccable (`setup toolbox`, then `run <command>`).
# The image tag is a hash of this folder and that script, so editing a pin rebuilds it.
ARG UV_IMAGE=ghcr.io/astral-sh/uv:0.12.20-python3.14-trixie-slim@sha256:6ee77524d145f48dcdede4d64411334bb426a0ad9835009215a69ff17e4d0e7e
FROM ${UV_IMAGE}

ARG UV_VERSION=0.12.20
ARG PYTHONS="3.12.14 3.13.15 3.14.7 3.14.7t"
# Resolve every tool's own dependencies as of this instant, so a rebuild resolves the same set.
ARG EXCLUDE_NEWER=2026-09-29T00:00:00Z
ARG TLA_VERSION=1.7.4
ARG TLA_SHA256=936a262061c914694dfd669a543be24573c45d5aa0ff20a8b96b23d01e050e88
ARG OSV_SCANNER_VERSION=2.6.0
ARG OSV_SCANNER_SHA256_AMD64=ca69b3d3cd08f889a49dc0a383122f71cc528b83803671df5fd874d97485b108
ARG OSV_SCANNER_SHA256_ARM64=2c71403eb443d05891c4f268c3ad771cf4f16e5443463fd7851ef8f454d3c7e4
# Atheris 3.1.0 has no git tag, no sdist, and PyPI wheels only for x86_64, so it is built
# here from the commit PyPI 3.1.0 was released from, for each interpreter it supports.
ARG ATHERIS_VERSION=3.1.0
ARG ATHERIS_COMMIT=e36da74cf42b5834c4e31d7cabb58d796695d29f
ARG ATHERIS_PYTHONS="3.12 3.13 3.14"

# gcc and g++ rebuild C and C++ extensions under ASan and UBSan. clang and its compiler-rt
# build Atheris against libFuzzer. Valgrind checks native memory. The JRE runs TLC, and
# Node runs Pyright.
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl git gcc g++ libc6-dev \
      clang libclang-rt-19-dev valgrind default-jre-headless nodejs \
 && rm -rf /var/lib/apt/lists/* \
 && git config --system --add safe.directory '*'

# Pinned CPython builds managed by uv, including the free-threaded 3.14t. After this
# layer uv never downloads an interpreter, so a missing version fails instead of
# silently fetching one. Pick one per run with UV_PYTHON (for example UV_PYTHON=3.14t).
ENV UV_PYTHON_INSTALL_DIR=/opt/uv-python \
    UV_TOOL_DIR=/opt/uv-tools \
    UV_TOOL_BIN_DIR=/usr/local/bin \
    UV_PYTHON_PREFERENCE=only-managed \
    UV_LINK_MODE=copy
RUN test "$(uv --version | cut -d' ' -f2)" = "${UV_VERSION}" \
 && uv python install --no-bin ${PYTHONS}
ENV UV_PYTHON_DOWNLOADS=never

RUN curl -fsSL -o /opt/tla2tools.jar "https://github.com/tlaplus/tlaplus/releases/download/v${TLA_VERSION}/tla2tools.jar" \
 && echo "${TLA_SHA256}  /opt/tla2tools.jar" | sha256sum -c - \
 && printf '#!/bin/sh\nexec java -XX:+UseParallelGC -cp /opt/tla2tools.jar tlc2.TLC "$@"\n' > /usr/local/bin/tlc \
 && chmod +x /usr/local/bin/tlc

RUN arch="$(dpkg --print-architecture)" \
 && case "${arch}" in amd64) sum="${OSV_SCANNER_SHA256_AMD64}" ;; arm64) sum="${OSV_SCANNER_SHA256_ARM64}" ;; *) exit 1 ;; esac \
 && curl -fsSL -o /usr/local/bin/osv-scanner \
      "https://github.com/google/osv-scanner/releases/download/v${OSV_SCANNER_VERSION}/osv-scanner_linux_${arch}" \
 && echo "${sum}  /usr/local/bin/osv-scanner" | sha256sum -c - \
 && chmod +x /usr/local/bin/osv-scanner

# `impeccable fuzz` installs the matching wheel into the project environment from here.
RUN git clone --quiet --filter=blob:none https://github.com/google/atheris /tmp/atheris \
 && git -C /tmp/atheris checkout --quiet "${ATHERIS_COMMIT}" \
 && for py in ${ATHERIS_PYTHONS}; do \
      uv venv --quiet --python "${py}" "/tmp/build-${py}" \
   && uv pip install --quiet --python "/tmp/build-${py}/bin/python" --exclude-newer "${EXCLUDE_NEWER}" setuptools wheel pybind11 \
   && (cd /tmp/atheris && ATHERIS_VERSION="${ATHERIS_VERSION}" CLANG_BIN=/usr/bin/clang CC=clang CXX=clang++ \
        "/tmp/build-${py}/bin/python" setup.py --quiet bdist_wheel --dist-dir /opt/impeccable/wheels) \
   && rm -rf /tmp/atheris/build "/tmp/build-${py}" || exit 1; \
    done \
 && rm -rf /tmp/atheris \
 && ls /opt/impeccable/wheels

COPY toolbox/tools.txt /usr/local/share/impeccable/tools.txt
RUN grep -v -E '^[[:space:]]*(#|$)' /usr/local/share/impeccable/tools.txt \
  | awk '$3 == "tool" { print $1 "==" $2 }' \
  | while read -r spec; do \
      uv tool install --quiet --python 3.14 --exclude-newer "${EXCLUDE_NEWER}" "${spec}" || exit 1; \
    done \
 && uv cache clean

COPY toolbox/Dockerfile toolbox/valgrind-python.supp /usr/local/share/impeccable/
COPY toolbox/entrypoint /usr/local/bin/impeccable-entrypoint
COPY scripts/impeccable /usr/local/bin/impeccable

# The project environment lives in a per-project /venvs volume and the uv cache in /cache,
# so the host's .venv is never touched and a rebuild of the image keeps both.
ENV IMPECCABLE_IN_TOOLBOX=1 \
    IMPECCABLE_WHEELS=/opt/impeccable/wheels \
    UV_CACHE_DIR=/cache/uv \
    XDG_CACHE_HOME=/cache/xdg
WORKDIR /work
ENTRYPOINT ["impeccable-entrypoint"]
CMD ["bash"]
