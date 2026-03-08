FROM ubuntu:24.04

# Base packages + SSH
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        openssh-server curl ca-certificates git sudo \
        build-essential procps file \
        zsh neovim \
    && mkdir -p /run/sshd \
    && sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config \
    && sed -i 's|^#*AuthorizedKeysFile.*|AuthorizedKeysFile .ssh/authorized_keys /etc/ssh/github_authorized_keys|' /etc/ssh/sshd_config \
    && curl -fsSL https://github.com/hiroyaonoe.keys > /etc/ssh/github_authorized_keys \
    && useradd -m -s /usr/bin/zsh -G sudo onoe \
    && echo 'onoe ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Homebrew
RUN useradd -m -s /bin/bash linuxbrew \
    && echo 'linuxbrew ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers \
    && NONINTERACTIVE=1 su -l linuxbrew -c \
        '/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"'
ENV PATH="/home/linuxbrew/.linuxbrew/bin:/home/linuxbrew/.linuxbrew/sbin:${PATH}"
RUN su -l linuxbrew -c 'brew install nodenv gh'

# Go (from go.dev)
ENV GOROOT=/usr/local/go PATH="/usr/local/go/bin:${PATH}"
RUN ARCH="$(dpkg --print-architecture)" \
    && GO_VERSION="$(curl -s 'https://go.dev/dl/?mode=json' | sed -n 's/.*"version": *"\(go[0-9][^"]*\)".*/\1/p' | head -1)" \
    && curl -fsSL "https://go.dev/dl/${GO_VERSION}.linux-${ARCH}.tar.gz" | tar -C /usr/local -xz

# Go tools (fzf, ghq, git-wt)
ENV GOBIN=/usr/local/bin
RUN go install github.com/junegunn/fzf@latest \
    && go install github.com/x-motemen/ghq@latest \
    && go install github.com/k1LoW/git-wt@latest \
    && rm -rf /root/go

# Node.js (via brew-installed nodenv)
ENV NODENV_ROOT=/opt/nodenv PATH="/opt/nodenv/shims:${PATH}"
RUN NODE_VERSION="$(nodenv install --list | grep -E '^\s*[0-9]+\.[0-9]+\.[0-9]+$' | grep -E '^\s*(18|20|22)\.' | tail -1 | tr -d ' ')" \
    && nodenv install "$NODE_VERSION" \
    && nodenv global "$NODE_VERSION" \
    && nodenv rehash

# uv + Python
ENV UV_PYTHON_INSTALL_DIR=/opt/uv/python
RUN curl -LsSf https://astral.sh/uv/install.sh | sh \
    && mv /root/.local/bin/uv /usr/local/bin/ \
    && mv /root/.local/bin/uvx /usr/local/bin/ \
    && rm -rf /root/.local \
    && uv python install --default \
    && ln -sf "$(uv python find)" /usr/local/bin/python3

# Claude Code
RUN curl -fsSL https://claude.ai/install.sh | bash

# Default shell
RUN chsh -s "$(which zsh)"

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D", "-e"]
