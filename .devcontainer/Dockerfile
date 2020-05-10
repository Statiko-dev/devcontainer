# Based on:
# https://github.com/nodejs/docker-node/blob/c31a071c73c5cc40dc662b75a4ee9f9fc23d6a39/12/buster/Dockerfile
# https://github.com/docker-library/golang/blob/f30f0428221b94c7dcb414ebdcea83106df20897/1.13/buster/Dockerfile
# https://github.com/microsoft/vscode-remote-try-node/blob/master/.devcontainer/Dockerfile
# https://github.com/microsoft/vscode-remote-try-go/blob/master/.devcontainer/Dockerfile

FROM ubuntu:18.04

# Versions
ENV NODE_VERSION 14.2.0
ENV GOLANG_VERSION 1.14.2
ENV ETCD_VERSION 3.4.7

# Avoid warnings by switching to noninteractive
ENV DEBIAN_FRONTEND=noninteractive

# Create a "dev" user
ARG USERNAME=dev
ENV USER_UID=1000
ENV USER_GID=1000
RUN groupadd --gid ${USER_GID} ${USERNAME} \
  && useradd --uid ${USER_UID} --gid ${USERNAME} --shell /bin/bash --create-home ${USERNAME} \
  && mkdir -p /home/${USERNAME}/.vscode-server /home/${USERNAME}/.vscode-server-insiders \
  && chown ${USER_UID}:${USER_GID} /home/${USERNAME}/.vscode-server*

# Configure apt and install packages
RUN apt-get update \
    && apt-get -y install --no-install-recommends apt-utils dialog 2>&1 \ 
    #
    # Verify git, GPG, process tools, lsb-release (common in install instructions for CLIs) installed, and add other tools
    # Add also nginx
    && apt-get -y install nginx git gnupg2 procps lsb-release \
        ca-certificates nano vim tree curl jq screen net-tools dsniff \
    #
    # Install Node.js
    && curl -fsSLO --compressed "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.gz" \
    && tar -xzf "node-v$NODE_VERSION-linux-x64.tar.gz" -C /usr/local --strip-components=1 --no-same-owner \
    && rm "node-v$NODE_VERSION-linux-x64.tar.gz" \
    && ln -s /usr/local/bin/node /usr/local/bin/nodejs \
    #
    # Install eslint globally
    && npm install -g eslint \
    #
    # Add sudo support for non-root users
    && apt-get install -y sudo \
    && echo ${USERNAME} ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/${USERNAME} \
    && chmod 0440 /etc/sudoers.d/${USERNAME} \
    #
    # Add ZSH and oh-my-zsh
    && apt-get -y install zsh \
    && wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | zsh || true \
    #
    # GCC for cgo
    && apt-get install -y --no-install-recommends \
        g++ \
        gcc \
        libc6-dev \
        make \
        pkg-config \
    #
    # Golang
    && curl -L -o go.tgz "https://golang.org/dl/go${GOLANG_VERSION}.linux-amd64.tar.gz" \
    && tar -C /usr/local -xzf go.tgz \
    && rm go.tgz \
    && export PATH="/usr/local/go/bin:$PATH" \
    && go version \
    #
    # Install etcd (mostly for etcdctl)
    && curl -fsSLO --compressed "https://github.com/etcd-io/etcd/releases/download/v${ETCD_VERSION}/etcd-v${ETCD_VERSION}-linux-amd64.tar.gz" \
    && tar xzf etcd-v${ETCD_VERSION}-linux-amd64.tar.gz \
    && mv etcd-v${ETCD_VERSION}-linux-amd64/etcd* /usr/local/bin/ \
    && rm -rf etc-v${ETCD_VERSION}-linux-amd64* \
    #
    # CA Certificate for Statiko tests
    && curl -fsSL "https://raw.githubusercontent.com/ItalyPaleAle/statiko/master/.e2e-test/misc/ca.crt" -o /usr/local/share/ca-certificates/italypaleale-ci.crt \
    && update-ca-certificates \
    #
    # Clean up
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# Environmental variables for go
ENV GOPATH /home/${USERNAME}/go
ENV PATH $GOPATH/bin:/usr/local/go/bin:$PATH

# Go tools for VS Code - require root
# Build Go tools w/module support
RUN mkdir -p /tmp/gotools \
    && cd /tmp/gotools \
    && GOPATH=/tmp/gotools GO111MODULE=on go get -v golang.org/x/tools/gopls@latest 2>&1 \
    && GOPATH=/tmp/gotools GO111MODULE=on go get -v \
        honnef.co/go/tools/...@latest \
        golang.org/x/tools/cmd/gorename@latest \
        golang.org/x/tools/cmd/goimports@latest \
        golang.org/x/tools/cmd/guru@latest \
        golang.org/x/lint/golint@latest \
        github.com/mdempsky/gocode@latest \
        github.com/cweill/gotests/...@latest \
        github.com/haya14busa/goplay/cmd/goplay@latest \
        github.com/sqs/goreturns@latest \
        github.com/josharian/impl@latest \
        github.com/davidrjenni/reftools/cmd/fillstruct@latest \
        github.com/uudashr/gopkgs/v2/cmd/gopkgs@latest  \
        github.com/ramya-rao-a/go-outline@latest  \
        github.com/acroca/go-symbols@latest  \
        github.com/godoctor/godoctor@latest  \
        github.com/rogpeppe/godef@latest  \
        github.com/zmb3/gogetdoc@latest \
        github.com/fatih/gomodifytags@latest  \
        github.com/mgechev/revive@latest  \
        github.com/golangci/golangci-lint/cmd/golangci-lint@v1.26.0  \
        github.com/go-delve/delve/cmd/dlv@latest \
    2>&1 \
    #
    # Build Go tools w/o module support
    && GOPATH=/tmp/gotools go get -v github.com/alecthomas/gometalinter 2>&1 \
    #
    # Build gocode-gomod
    && GOPATH=/tmp/gotools go get -x -d github.com/stamblerre/gocode 2>&1 \
    && GOPATH=/tmp/gotools go build -o gocode-gomod github.com/stamblerre/gocode \
    #
    # Install Go tools
    && mv /tmp/gotools/bin/* /usr/local/bin/ \
    && mv gocode-gomod /usr/local/bin/

# Workspace
RUN mkdir /workspace \
    && chown ${USER_UID}:${USER_GID} /workspace
WORKDIR /workspace

# Set username
USER ${USERNAME}

# Enable Go modules
ENV GO111MODULE=on

# Go tools for Statiko
RUN mkdir -p ${GOPATH}/bin ${GOPATH}/src ${GOPATH}/pkg \
    && cd ${GOPATH} \
    && GO111MODULE=off go get -u -v \
        github.com/axw/gocov/gocov \
        gopkg.in/matm/v1/gocov-html \
        github.com/gobuffalo/packr/v2/packr2 \
        2>&1

# Other environmental variables
ENV ETCDCTL_API 3

# Switch back to dialog for any ad-hoc use of apt-get
ENV DEBIAN_FRONTEND=

CMD ["zsh"]
