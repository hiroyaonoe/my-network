FROM ubuntu:24.04

# System packages (rarely changes)
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        openssh-server curl ca-certificates git sudo \
        build-essential procps file \
        zsh neovim tmux \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# System config + users (rarely changes)
RUN mkdir -p /run/sshd \
    && sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config \
    && sed -i 's|^#*AuthorizedKeysFile.*|AuthorizedKeysFile .ssh/authorized_keys /etc/ssh/github_authorized_keys|' /etc/ssh/sshd_config \
    && useradd -m -s /usr/bin/zsh -G sudo onoe \
    && echo 'onoe ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers \
    && chsh -s /usr/bin/zsh

# Homebrew (rarely changes)
USER onoe
RUN NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
ENV PATH="/home/linuxbrew/.linuxbrew/bin:/home/linuxbrew/.linuxbrew/sbin:${PATH}"

# Brew packages (update ARG to rebuild)
ARG BREW_PACKAGES="nodenv gh"
RUN brew install ${BREW_PACKAGES}

# Go (update ARG to rebuild)
ARG GO_VERSION=latest
ENV GOROOT=/usr/local/go PATH="/usr/local/go/bin:${PATH}"
USER root
RUN if [ "${GO_VERSION}" = "latest" ]; then \
        GO_VERSION="$(curl -s 'https://go.dev/dl/?mode=json' | sed -n 's/.*"version": *"\(go[0-9][^"]*\)".*/\1/p' | head -1)"; \
    fi \
    && ARCH="$(dpkg --print-architecture)" \
    && curl -fsSL "https://go.dev/dl/${GO_VERSION}.linux-${ARCH}.tar.gz" | tar -C /usr/local -xz

# Go tools
ENV GOBIN=/usr/local/bin
RUN go install github.com/junegunn/fzf@latest \
    && go install github.com/x-motemen/ghq@latest \
    && go install github.com/k1LoW/git-wt@latest \
    && rm -rf /root/go

# Node.js via nodenv (update ARG to rebuild)
ARG NODE_MAJOR=22
ENV NODENV_ROOT=/opt/nodenv PATH="/opt/nodenv/shims:${PATH}"
RUN mkdir -p /opt/nodenv && chown onoe:onoe /opt/nodenv
USER onoe
RUN NODE_VERSION="$(nodenv install --list | grep -E '^\s*[0-9]+\.[0-9]+\.[0-9]+$' | grep -E "^\s*${NODE_MAJOR}\." | tail -1 | tr -d ' ')" \
    && nodenv install "$NODE_VERSION" \
    && nodenv global "$NODE_VERSION" \
    && nodenv rehash

# uv + Python
USER root
ENV UV_PYTHON_INSTALL_DIR=/opt/uv/python
RUN curl -LsSf https://astral.sh/uv/install.sh | sh \
    && mv /root/.local/bin/uv /usr/local/bin/ \
    && mv /root/.local/bin/uvx /usr/local/bin/ \
    && rm -rf /root/.local \
    && uv python install --default \
    && ln -sf "$(uv python find)" /usr/local/bin/python3 \
    && ln -sf "$(uv python find)" /usr/local/bin/python

# Claude Code
USER onoe
RUN curl -fsSL https://claude.ai/install.sh | bash

# GitHub SSH keys (changes when keys rotate)
USER root
RUN curl -fsSL https://github.com/hiroyaonoe.keys > /etc/ssh/github_authorized_keys

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D", "-e"]
