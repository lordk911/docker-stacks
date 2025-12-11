# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.
ARG REGISTRY=quay.io
ARG OWNER=jupyter
ARG BASE_IMAGE=$REGISTRY/$OWNER/scipy-notebook:python-3.11
FROM $BASE_IMAGE

LABEL maintainer="Jupyter Project <jupyter@googlegroups.com>"

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# Spark dependencies
# Default values can be overridden at build time
# (ARGS are in lowercase to distinguish them from ENV)
ARG openjdk_version="17"

ENV TZ=Asia/Shanghai

#RUN printf "deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble main restricted universe multiverse\n\
#deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-updates main restricted universe multiverse\n\
#deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-backports main restricted universe multiverse\n\
#deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-security main restricted universe multiverse\n" \
#> /etc/apt/sources.list

RUN printf "channels:\n  - defaults\nshow_channel_urls: true\ndefault_channels:\n  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main\n  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r\ncustom_channels:\n  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud\n  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud\n  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud\n  menpo: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud\n  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud\n  pytorch-lts: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud\n  simpleitk: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud\n" > /root/.condarc
RUN pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    "openjdk-${openjdk_version}-jre-headless" \
    ca-certificates-java && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# If spark_version is not set, latest Spark will be installed
ARG spark_version
ARG hadoop_version="3"
# If scala_version is not set, Spark without Scala will be installed
ARG scala_version
# URL to use for Spark downloads
# You need to use https://archive.apache.org/dist/spark/ website if you want to download old Spark versions
# But it seems to be slower, that's why we use the recommended site for download
ARG spark_download_url="https://dlcdn.apache.org/spark/"

ENV SPARK_HOME=/usr/local/spark
ENV PATH="${PATH}:${SPARK_HOME}/bin"
ENV SPARK_OPTS="--driver-java-options=-Xms1024M --driver-java-options=-Xmx4096M --driver-java-options=-Dlog4j.logLevel=info"

COPY setup_spark.py /opt/setup-scripts/

# Setup Spark
RUN /opt/setup-scripts/setup_spark.py \
    --spark-version="${spark_version}" \
    --hadoop-version="${hadoop_version}" \
    --scala-version="${scala_version}" \
    --spark-download-url="${spark_download_url}"

# Configure IPython system-wide
COPY ipython_kernel_config.py "/etc/ipython/"
RUN fix-permissions "/etc/ipython/"

# macOS Rosetta virtualization creates junk directory which gets owned by root further up.
# It'll get re-created, but as USER runner after the next directive so hopefully should not cause permission issues.
#
# More info: https://github.com/jupyter/docker-stacks/issues/2296
# hadolint ignore=DL3059
RUN rm -rf "/home/${NB_USER}/.cache/"

USER ${NB_UID}

# Install pyarrow
# NOTE: It's important to ensure compatibility between Pandas versions.
# The pandas version in this Dockerfile should match the version
# on which the Pandas API for Spark is built.
# To find the right version, check the pandas version being installed here:
# https://github.com/apache/spark/blob/<SPARK_VERSION>/dev/infra/Dockerfile
RUN mamba install --yes \
    'grpcio-status' \
    'grpcio' \
    'pandas=2.2.3' \
    'pyarrow' && \
    pip install \
    'notebook' \
    'jupyter-collaboration' \
    'jupyter-mcp-server>=0.15.0' \
    'datalayer_pycrdt>=0.12.17' \
    'jupyter_mcp_tools' \
    'jupyter-ai' \
    'langchain-anthropic' \
    'langchain-openai' && \
    mamba clean --all -f -y

USER root

RUN rm -rf "/home/${NB_USER}/.cache/"
# 执行权限修复操作
# 确保 fix-permissions 脚本以 root 权限执行，以成功使用 chgrp/chmod
RUN fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

# 步骤 3: 切换回非 root 用户作为最终运行用户
USER ${NB_UID}

WORKDIR "${HOME}"
EXPOSE 4040
