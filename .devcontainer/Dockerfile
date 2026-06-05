# Using the latest Ubuntu 26.04 LTS (Resolute Raccoon) base image
FROM docker.io/ubuntu:resolute

# Prevent interactive prompts during installation
ENV DEBIAN_FRONTEND=noninteractive

# Install core build tools, C++ dependencies, and Node.js
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    ca-certificates \
    clang \
    clang-format \
    clang-tidy \
    clangd \
    cmake \
    curl \
    fd-find \
    gdb \
    git \
    gnupg \
    less \
    lld \
    ninja-build \
    nodejs \
    npm \
    ripgrep \
    sudo \
    zsh \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install the pi.dev agent CLI globally using npm
RUN npm install -g --ignore-scripts @earendil-works/pi-coding-agent

# Set up user
ARG USERNAME=ubuntu
RUN usermod -aG sudo ${USERNAME} \
    && echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers \
    && chsh -s /bin/zsh ${USERNAME} \
    && cp /etc/zsh/newuser.zshrc.recommended /home/${USERNAME}/.zshrc \
    && chown ${USERNAME}:${USERNAME} /home/${USERNAME}/.zshrc
USER ${USERNAME}

RUN pi install npm:pi-intercom \
    && pi install npm:pi-subagents \
    && pi install npm:pi-web-access
