FROM ubuntu:24.04

# Layer 1: System packages (rarely changes)
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        openssh-server curl ca-certificates git sudo \
        build-essential procps file \
        zsh neovim \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Layer 2: System config + users (rarely changes)
RUN mkdir -p /run/sshd \
    && sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config \
    && sed -i 's|^#*AuthorizedKeysFile.*|AuthorizedKeysFile .ssh/authorized_keys /etc/ssh/github_authorized_keys|' /etc/ssh/sshd_config \
    && useradd -m -s /usr/bin/zsh -G sudo onoe \
    && echo 'onoe ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers \
    && chsh -s /usr/bin/zsh

# Layer 3: Homebrew (rarely changes)
RUN useradd -m -s /bin/bash linuxbrew \
    && echo 'linuxbrew ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers \
    && NONINTERACTIVE=1 su -l linuxbrew -c \
        '/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"'
ENV PATH="/home/linuxbrew/.linuxbrew/bin:/home/linuxbrew/.linuxbrew/sbin:${PATH}"

# Layer 4: brew packages (update ARG to rebuild)
ARG BREW_PACKAGES="nodenv gh"
RUN su -l linuxbrew -c "brew install ${BREW_PACKAGES}"

# Layer 5: Go (update ARG to rebuild)
ARG GO_VERSION=latest
ENV GOROOT=/usr/local/go PATH="/usr/local/go/bin:${PATH}"
RUN if [ "${GO_VERSION}" = "latest" ]; then \
        GO_VERSION="$(curl -s 'https://go.dev/dl/?mode=json' | sed -n 's/.*"version": *"\(go[0-9][^"]*\)".*/\1/p' | head -1)"; \
    fi \
    && ARCH="$(dpkg --print-architecture)" \
    && curl -fsSL "https://go.dev/dl/${GO_VERSION}.linux-${ARCH}.tar.gz" | tar -C /usr/local -xz

# Layer 6: Go tools
ENV GOBIN=/usr/local/bin
RUN go install github.com/junegunn/fzf@latest \
    && go install github.com/x-motemen/ghq@latest \
    && go install github.com/k1LoW/git-wt@latest \
    && rm -rf /root/go

# Layer 7: Node.js via nodenv (update ARG to rebuild)
ARG NODE_MAJOR=22
ENV NODENV_ROOT=/opt/nodenv PATH="/opt/nodenv/shims:${PATH}"
RUN NODE_VERSION="$(nodenv install --list | grep -E '^\s*[0-9]+\.[0-9]+\.[0-9]+$' | grep -E "^\s*${NODE_MAJOR}\." | tail -1 | tr -d ' ')" \
    && nodenv install "$NODE_VERSION" \
    && nodenv global "$NODE_VERSION" \
    && nodenv rehash

# Layer 8: uv + Python
ENV UV_PYTHON_INSTALL_DIR=/opt/uv/python
RUN curl -LsSf https://astral.sh/uv/install.sh | sh \
    && mv /root/.local/bin/uv /usr/local/bin/ \
    && mv /root/.local/bin/uvx /usr/local/bin/ \
    && rm -rf /root/.local \
    && uv python install --default \
    && ln -sf "$(uv python find)" /usr/local/bin/python3

# Layer 9: Claude Code
RUN curl -fsSL https://claude.ai/install.sh | bash

# Layer 10: GitHub SSH keys (changes when keys rotate)
RUN curl -fsSL https://github.com/hiroyaonoe.keys > /etc/ssh/github_authorized_keys

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D", "-e"]
