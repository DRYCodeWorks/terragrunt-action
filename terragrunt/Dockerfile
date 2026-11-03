# Container to run Terragrunt and Terraform
# Contains inside TFenv and TGSwitch to allow users to install custom Terraform and Terragrunt versions

# Node builds Node Lambdas inside Terraform, as Python's packager does for
# Python ones. Keep it on the Lambda runtime's major version.
ARG NODE_IMAGE=node:24-bookworm-slim
ARG UV_IMAGE=ghcr.io/astral-sh/uv:0.13.7
FROM ${NODE_IMAGE} AS node
FROM ${UV_IMAGE} AS uv

# The latest Ubuntu LTS. Python doesn't come from Ubuntu: packagers
# pip-install for the Lambda runtime with whichever python3 is on PATH, so
# the image installs the versions the runtimes need (see PYTHON_VERSIONS).
FROM ubuntu:26.04

ARG TF_ENV_VERSION=v3.0.0
ARG TGSWITCH_VERSION=0.6.0
# Python versions to install, space-separated; the first is `python3` and
# `pip`. It must match the Python Lambda runtime (python3.12).
ARG PYTHON_VERSIONS="3.12"

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y \
    ca-certificates \
    curl \
    file \
    git \
    jq \
    unzip \
    wget \
    sudo \
    rsync \
    zip \
    && rm -rf /var/lib/apt/lists/*

# AWS CLI v2 from AWS's installer (it bundles its own Python). Packagers use
# v2-only commands such as `aws configure export-credentials`.
RUN curl -fsSL https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o /tmp/awscliv2.zip \
    && unzip -q /tmp/awscliv2.zip -d /tmp \
    && /tmp/aws/install \
    && rm -rf /tmp/aws /tmp/awscliv2.zip

# Python from uv's standalone builds, readable by the runner user.
COPY --from=uv /uv /usr/local/bin/uv
ENV UV_PYTHON_INSTALL_DIR=/opt/python
RUN uv python install ${PYTHON_VERSIONS} \
    && for v in ${PYTHON_VERSIONS}; do ln -s "$(uv python find "$v")" "/usr/local/bin/python$v"; done \
    && default="$(echo ${PYTHON_VERSIONS} | cut -d' ' -f1)" \
    && ln -s "python$default" /usr/local/bin/python3 \
    && ln -s python3 /usr/local/bin/python \
    && printf '#!/bin/sh\nexec python3 -m pip "$@"\n' > /usr/local/bin/pip \
    && chmod 755 /usr/local/bin/pip \
    && ln -s pip /usr/local/bin/pip3 \
    && python3 -m pip --version \
    && chmod -R a+rX /opt/python

COPY --from=node /usr/local/bin/node /usr/local/bin/node
COPY --from=node /usr/local/lib/node_modules /usr/local/lib/node_modules
RUN ln -s ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
    && ln -s ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx

# Create runner user
RUN addgroup --system --gid 127 docker
RUN useradd --system -u 1001 -g 127 -ms /bin/bash runner
RUN usermod -aG sudo runner && echo 'runner ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
USER runner

RUN mkdir -p /home/runner/.ssh
COPY ./known_hosts /home/runner/.ssh/known_hosts

# clone tfenv
RUN git clone --depth=1 --branch ${TF_ENV_VERSION} https://github.com/tfutils/tfenv.git ~/.tfenv
RUN echo 'export PATH="${HOME}/.tfenv/bin:${PATH}"' >> ~/.bash_profile

# install tgswitch
RUN mkdir -p "${HOME}/tgswitch"
RUN wget -q https://github.com/warrensbox/tgswitch/releases/download/${TGSWITCH_VERSION}/tgswitch_${TGSWITCH_VERSION}_linux_amd64.tar.gz -O /tmp/tgswitch_${TGSWITCH_VERSION}_linux_amd64.tar.gz
RUN tar -xzf /tmp/tgswitch_${TGSWITCH_VERSION}_linux_amd64.tar.gz -C ${HOME}/tgswitch
RUN chmod u+x ${HOME}/tgswitch/tgswitch
RUN rm -rf /tmp/tgswitch_${TGSWITCH_VERSION}_linux_amd64.tar.gz

# Running action as runner user
# https://docs.github.com/en/actions/creating-actions/dockerfile-support-for-github-actions#user
ENV PATH="/home/runner/.tfenv/bin:/home/runner/tgswitch:/home/runner/bin:${PATH}"

ENV TF_INPUT=false
ENV TF_IN_AUTOMATION=1
