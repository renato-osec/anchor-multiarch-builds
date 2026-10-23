#
# Docker image to generate deterministic, verifiable builds of Anchor programs.
# This must be run *after* a given ANCHOR_CLI version is published and a git tag
# is released on GitHub.
#

FROM ubuntu:22.04@sha256:1ec65b2719518e27d4d25f104d93f9fac60dc437f81452302406825c46fcc9cb

ARG DEBIAN_FRONTEND=noninteractive

ARG SOLANA_CLI
ARG ANCHOR_CLI
ARG NODE_VERSION="v22.12.0"
ARG TARGETARCH

ENV HOME="/root"
ENV PATH="${HOME}/.cargo/bin:${PATH}"
ENV PATH="${HOME}/.local/share/solana/install/active_release/bin:${PATH}"
ENV PATH="${HOME}/.nvm/versions/node/${NODE_VERSION}/bin:${PATH}"

# Install base utilities.
RUN mkdir -p /workdir && mkdir -p /tmp && \
    apt-get update -qq && apt-get upgrade -qq && apt-get install -qq \
    build-essential git curl wget jq pkg-config python3-pip \
    libssl-dev libudev-dev

# Install rust.
RUN curl "https://sh.rustup.rs" -sfo rustup.sh && \
    sh rustup.sh -y && \
    rustup component add rustfmt clippy

# Install node / npm / yarn.
RUN curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.11/install.sh | bash
ENV NVM_DIR="${HOME}/.nvm"
RUN . $NVM_DIR/nvm.sh && \
    nvm install ${NODE_VERSION} && \
    nvm use ${NODE_VERSION} && \
    nvm alias default node && \
    npm install -g yarn

# Install Solana tools. Agave has no Linux arm64 release, so arm64 gets only the
# `cargo-build-sbf` that SOLANA_CLI pins and the platform-tools that the CLI's
# BUILD_SUBCOMMAND pins, with the x86_64 package's SBF sysroot: the two packages
# ship differently built standard libraries, which changes every program.
RUN if [ "${TARGETARCH}" = "arm64" ]; then \
        cargo install --locked cargo-build-sbf --version 4.1.0 && \
        tools="${HOME}/.cache/solana/v1.52/platform-tools" && \
        url="https://github.com/anza-xyz/platform-tools/releases/download/v1.52" && \
        mkdir -p "${tools}" && \
        curl -sSfL "${url}/platform-tools-linux-aarch64.tar.bz2" | tar xj -C "${tools}" && \
        rm -rf "${tools}"/rust/lib/rustlib/sb*-solana-solana && \
        curl -sSfL "${url}/platform-tools-linux-x86_64.tar.bz2" | \
            tar xj -C "${tools}" --wildcards './rust/lib/rustlib/sb*-solana-solana'; \
    else \
        sh -c "$(curl -sSfL https://release.anza.xyz/${SOLANA_CLI}/install)"; \
    fi

# Install anchor.
RUN cargo install --git https://github.com/otter-sec/anchor --tag ${ANCHOR_CLI} anchor-cli --locked

# Make SBF builds independent of the image architecture.
COPY sbf-rustc-wrapper /usr/local/bin/sbf-rustc-wrapper
ENV RUSTC_WRAPPER="/usr/local/bin/sbf-rustc-wrapper"

# Build a dummy program to bootstrap the BPF SDK (doing this speeds up builds).
RUN mkdir -p /tmp && cd tmp && anchor init dummy && cd dummy && anchor build --no-idl

WORKDIR /workdir
