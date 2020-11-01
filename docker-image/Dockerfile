FROM ubuntu:20.04

# Upgrade and install curl, sudo and essential development tools
RUN apt update -y && apt upgrade -y && \
    apt install -y curl git sudo build-essential lldb

# Add a user `rustdev` so that you're not developing as the `root` user
# The user needs to sudoer be able to install code-server
RUN adduser --gecos '/usr/bin/bash' --disabled-password rustdev && \
  echo "rustdev ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/nopasswd
ENV SHELL bash

USER rustdev
WORKDIR /home/rustdev
# Create USER environment variable to prevent "could not determine the current user, please set $USER" error when running "cargo new ..."
ENV USER rustdev

# Install rustup
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs > rustup.sh
RUN sh rustup.sh -y && rm -f rustup.sh
ENV PATH /home/rustdev/.cargo/bin:$PATH

# Install code-server 
RUN curl -fsSL https://code-server.dev/install.sh > install.sh
RUN sh install.sh --method=standalone
RUN rm -f install.sh
RUN sudo chown -R rustdev:rustdev /home/rustdev/.local
ENV PATH /home/rustdev/.local/bin:$PATH

# Install code-server extensions

## Install extensions dependencies
RUN rustup component add rust-analysis --toolchain stable-x86_64-unknown-linux-gnu
RUN rustup component add rust-src --toolchain stable-x86_64-unknown-linux-gnu
RUN rustup component add rls --toolchain stable-x86_64-unknown-linux-gnu

## Adds language support for Rust
RUN code-server --install-extension rust-lang.rust

## Alternative Rust Language Server
RUN code-server --install-extension matklad.rust-analyzer
### Download and Install rus-analyzer binary
RUN mkdir -p /home/rustdev/.local/share/code-server/User/globalStorage/matklad.rust-analyzer/
RUN curl -L https://github.com/rust-analyzer/rust-analyzer/releases/latest/download/rust-analyzer-linux -o /home/rustdev/.local/share/code-server/User/globalStorage/matklad.rust-analyzer/rust-analyzer-linux
RUN chmod u+x /home/rustdev/.local/share/code-server/User/globalStorage/matklad.rust-analyzer/rust-analyzer-linux

## Debugging tools
## Install from the vsix instead of using extension id because of this issue: https://github.com/vadimcn/vscode-lldb/issues/314
RUN curl -L https://github.com/vadimcn/vscode-lldb/releases/download/v1.6.0/codelldb-x86_64-linux.vsix -o codelldb-x86_64-linux.vsix
RUN code-server --install-extension codelldb-x86_64-linux.vsix
RUN rm -f codelldb-x86_64-linux.vsix

## Toml extension, usefull to edit cargo configuration files
RUN code-server --install-extension bungcip.better-toml

# Remove sudo package
USER root
ENV SUDO_FORCE_REMOVE yes
RUN apt -y remove sudo

# Run code server
USER rustdev

CMD ["code-server", "--bind-addr=0.0.0.0:8080", "--auth=none"]  