# yomi Olares app image: debian + yomi + basic tooling, nothing else.
# Every downloaded binary is version-pinned and sha256-verified.
# The image contains no configuration and no secrets.
# Optional toolchains (node, ...) are self-installed by the agent into the persistent
# home at runtime — curl/xz/tar below are all the bootstrap needs. See README.
FROM docker.io/library/debian:trixie-slim

# Mirror override for restricted networks: --build-arg APT_SOURCE=mirrors.ivolces.com
ARG APT_SOURCE=""
RUN if [ -n "${APT_SOURCE}" ]; then \
      sed -i "s|deb.debian.org|${APT_SOURCE}|g" /etc/apt/sources.list.d/debian.sources; \
    fi \
  && apt-get update \
  && apt-get install -y --no-install-recommends \
    bash \
    ca-certificates \
    curl \
    git \
    jq \
    openssh-client \
    python3 \
    ripgrep \
    tmux \
    xz-utils \
  && rm -rf /var/lib/apt/lists/*

# yomi itself (sha256 from the GitHub release asset digest;
# mirror: YOMI_DOWNLOAD_BASE=https://gh-proxy.com/https://github.com/Crescent617/yomi/releases/download)
ARG YOMI_VERSION=0.7.75
ARG YOMI_DOWNLOAD_BASE="https://github.com/Crescent617/yomi/releases/download"
ARG YOMI_LINUX_AMD64_SHA256=a052edc31343d3ee5df655300121ee0ddf8f9d711abc9c07482026ab8551f0aa
RUN set -eux; \
  yomi_url="${YOMI_DOWNLOAD_BASE}/v${YOMI_VERSION}/yomi-${YOMI_VERSION}-x86_64-unknown-linux-gnu.tar.gz"; \
  curl --http1.1 -fsSL --retry 5 --retry-all-errors --retry-delay 3 --connect-timeout 20 --max-time 600 -o /tmp/yomi.tar.gz "${yomi_url}"; \
  echo "${YOMI_LINUX_AMD64_SHA256}  /tmp/yomi.tar.gz" | sha256sum -c -; \
  tar -xzf /tmp/yomi.tar.gz -C /tmp; \
  install -m 0755 "/tmp/yomi-${YOMI_VERSION}-x86_64-unknown-linux-gnu/yomi" /usr/local/bin/yomi; \
  yomi version; \
  rm -rf /tmp/yomi.tar.gz "/tmp/yomi-${YOMI_VERSION}-x86_64-unknown-linux-gnu"

# The entire HOME directory is persisted on user storage (Olares Home/yomi):
# config, skills, sessions, OAuth tokens and caches all live on the volume.
ENV HOME=/home/yomi
RUN mkdir -p /home/yomi
WORKDIR /home/yomi

ENTRYPOINT ["yomi", "daemon", "start"]
