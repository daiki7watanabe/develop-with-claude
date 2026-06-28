FROM ubuntu:26.04

ARG TZ
ARG CLAUDE_CODE_VERSION=latest
ARG USERNAME=ubuntu

ENV TZ="$TZ"
ENV DEVCONTAINER=true

# 
RUN apt-get update && apt-get install -y --no-install-recommends \
  git \
  procps \
  fzf \
  zsh \
  man-db \
  unzip \
  gnupg2 \
  gh \
  sudo \
  iptables \
  ipset \
  iproute2 \
  dnsutils \
  aggregate \
  jq \
  curl \
  ca-certificates \
  && update-ca-certificates \
  && apt-get clean && rm -rf /var/lib/apt/lists/*


# Create workspace and config directories and set permissions
RUN mkdir -p /workspace /home/ubuntu/.claude && \
    chown -R ubuntu:ubuntu /workspace /home/ubuntu/.claude

WORKDIR /workspace

# Persist bash history.
RUN SNIPPET="export PROMPT_COMMAND='history -a' && export HISTFILE=/commandhistory/.bash_history" \
    && mkdir /commandhistory \
    && touch /commandhistory/.bash_history \
    && chown -R $USERNAME /commandhistory


# Install Claude
RUN curl -fsSL https://claude.ai/install.sh | bash -s ${CLAUDE_CODE_VERSION}


# Copy and set up firewall script
COPY init-firewall.sh /usr/local/bin/
USER root
RUN chmod +x /usr/local/bin/init-firewall.sh && \
  echo "ubuntu ALL=(root) NOPASSWD: /usr/local/bin/init-firewall.sh" > /etc/sudoers.d/ubuntu-firewall && \
  chmod 0440 /etc/sudoers.d/ubuntu-firewall
USER ubuntu
