FROM ubuntu:20.04

# 避免交互式安装提示
ENV DEBIAN_FRONTEND=noninteractive

# 安装 Ceph 编译所需的基础依赖
RUN apt-get update && apt-get install -y \
    git cmake make ccache g++ libssl-dev libboost-all-dev \
    libedit-dev libxml2-dev uuid-dev libaio-dev libsnappy-dev \
    liblz4-dev libcurl4-openssl-dev libkeyutils-dev libudev-dev \
    python3-dev python3-pip cython3 pkg-config libfmt-dev \
    libgoogle-perftools-dev libnss3-dev libbabeltrace-dev \
    libicu-dev libfuse-dev liblua5.3-dev libsqlite3-dev \
    libtool automake autoconf yasm nasm htop \
    && rm -rf /var/lib/apt/lists/*

RUN git clone https://github.com/axboe/liburing.git && \
    cd liburing && \
    ./configure && \
    make && \
    make install && \
    cd .. && rm -rf liburing

# 设置工作目录
WORKDIR /workspaces/ceph-fork
