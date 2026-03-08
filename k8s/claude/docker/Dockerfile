FROM ubuntu:24.04
RUN apt-get update \
    && apt-get install -y --no-install-recommends openssh-server curl ca-certificates \
    && apt-get clean && rm -rf /var/lib/apt/lists/* \
    && mkdir -p /run/sshd \
    && sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config \
    && sed -i 's|^#*AuthorizedKeysFile.*|AuthorizedKeysFile .ssh/authorized_keys /etc/ssh/github_authorized_keys|' /etc/ssh/sshd_config \
    && curl -fsSL https://github.com/hiroyaonoe.keys > /etc/ssh/github_authorized_keys
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D", "-e"]
