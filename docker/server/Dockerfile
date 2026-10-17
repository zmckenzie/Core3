# Core3 server image (used by ../../docker-compose.yml via ./swgemu).
#
# Stage 1 compiles engine3 + core3 from the checked-out source with clang and
# ccache (the ccache directory is a BuildKit cache mount, so rebuilds after a
# code change are incremental). Stage 2 is a slim runtime with only the
# shared libraries core3 links against.
#
# Ubuntu 24.04 is used because it ships clang-19 in its official repos,
# so the build does not depend on the third-party apt.llvm.org repository.

ARG UBUNTU_VERSION=24.04

FROM ubuntu:${UBUNTU_VERSION} AS build

ARG CLANG_VERSION=19
ENV DEBIAN_FRONTEND=noninteractive

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    set -eux; \
    rm -f /etc/apt/apt.conf.d/docker-clean; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        ccache \
        clang-${CLANG_VERSION} \
        cmake \
        default-jre-headless \
        g++ \
        git \
        libboost-thread-dev \
        libdb5.3-dev \
        libjemalloc-dev \
        liblua5.3-dev \
        libmariadb-dev \
        libmariadb-dev-compat \
        libssl-dev \
        lld-${CLANG_VERSION} \
        make \
        ninja-build \
        zlib1g-dev; \
    for tool in clang clang++ ld.lld; do \
        ln -sf "/usr/bin/${tool}-${CLANG_VERSION}" "/usr/bin/${tool}"; \
    done; \
    clang++ --version; \
    ld.lld --version

ENV CCACHE_DIR=/ccache \
    CCACHE_MAXSIZE=5G \
    CCACHE_BASEDIR=/core3 \
    CCACHE_COMPILERCHECK=content

# Must not contain "src/" anywhere above MMOCoreORB: src/CMakeLists.txt derives
# autogen paths with string(REPLACE "src/" ...) on the absolute path.
COPY MMOCoreORB /core3/MMOCoreORB
WORKDIR /core3/MMOCoreORB

# BUILD_JOBS empty = pick automatically from CPUs and memory. Each clang job
# compiling Core3 can use ~2GB, and Docker Desktop on WSL only gets half the
# machine's RAM by default, so -j$(nproc) alone tends to get OOM-killed.
ARG BUILD_TYPE=RelWithDebInfo
ARG BUILD_JOBS=
ARG CMAKE_EXTRA_ARGS=

RUN --mount=type=cache,target=/ccache \
    set -eu; \
    if [ ! -f utils/engine3/MMOEngine/CMakeLists.txt ]; then \
        echo "ERROR: MMOCoreORB/utils/engine3 is empty. Run: git submodule update --init --recursive" >&2; \
        exit 1; \
    fi; \
    jobs="${BUILD_JOBS}"; \
    if [ -z "${jobs}" ]; then \
        mem_gb=$(awk '/MemAvailable/ { print int($2 / 1024 / 1024) }' /proc/meminfo); \
        jobs=$(( mem_gb / 2 )); \
        [ "${jobs}" -gt "$(nproc)" ] && jobs=$(nproc); \
        [ "${jobs}" -lt 1 ] && jobs=1; \
        echo "Auto-selected ${jobs} build job(s) (cpus=$(nproc), available memory=${mem_gb}GB)"; \
    fi; \
    ccache -z; \
    cmake -G Ninja -S . -B /build \
        -DCMAKE_BUILD_TYPE="${BUILD_TYPE}" \
        -DENABLE_BUILD_CLIENT=OFF \
        -DRUN_GIT=OFF \
        ${CMAKE_EXTRA_ARGS}; \
    ninja -C /build -j "${jobs}" core3; \
    ccache -s; \
    test -x bin/core3

FROM ubuntu:${UBUNTU_VERSION} AS runtime

ENV DEBIAN_FRONTEND=noninteractive

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    set -eux; \
    rm -f /etc/apt/apt.conf.d/docker-clean; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        gdb \
        libatomic1 \
        libboost-thread1.83.0 \
        libdb5.3t64 \
        libjemalloc2 \
        liblua5.3-0 \
        libmariadb3 \
        libssl3t64 \
        mariadb-client \
        zlib1g; \
    useradd --create-home --uid 10001 --user-group --shell /bin/bash core3

COPY --from=build --chown=core3:core3 /core3/MMOCoreORB/bin /opt/core3/bin
COPY --chmod=0755 docker/server/entrypoint.sh /usr/local/bin/core3-entrypoint

# Fail the image build (not first boot) if a runtime library is missing.
RUN set -eu; \
    if ldd /opt/core3/bin/core3 | grep 'not found'; then exit 1; fi; \
    mkdir -p /tre /opt/core3/bin/databases /opt/core3/bin/log /opt/core3/bin/navmeshes /opt/core3/config.d; \
    chown core3:core3 /opt/core3/bin/databases /opt/core3/bin/log /opt/core3/bin/navmeshes

USER core3
WORKDIR /opt/core3/bin

# login, ping, zone (udp) and status (tcp)
EXPOSE 44453/udp 44462/udp 44463/udp 44455/tcp

HEALTHCHECK --interval=30s --timeout=5s --start-period=20m --retries=3 \
    CMD bash -c '</dev/tcp/127.0.0.1/44455' || exit 1

ENTRYPOINT ["core3-entrypoint"]
