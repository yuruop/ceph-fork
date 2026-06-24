此版本增加了.devcontainer配置脚本，推荐通过VS Code的Dev Containers插件使用

构建版本为：Ubuntu 20.04
Ceph Tag: v17.2.7-custom
Git Branch: cache

主要针对BlueStore层的Onode Cache的缓存替换算法做了S3FIFO的修改与适配
同时增加了客户端层面关于文件更细粒度的缓存命中统计

编译流程：
    cd ceph-fork
    # 切换pip清华源
    export PIP_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple
    ./install-deps.sh
    # 将整个 ceph-fork 目录及其所有子目录都标记为安全，因为docker环境下，用户可能不统一
    git config --global --add safe.directory '*'

    ./do_cmake.sh
    # do_cmake.sh不是release版本的编译，所以再重新cmake修改BUILD_TYPE
    cd build && cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo
    ninja -j20

    cd .. && dpkg-buildpackage -us -uc -b -j20

ninja过程中可能会有关于`node_modules package-lock.json`的报错
如果出现了再执行以下操作：
    source /workspaces/ceph-fork/build/src/pybind/mgr/dashboard/frontend/node-env/bin/activate
    cd /workspaces/ceph-fork/src/pybind/mgr/dashboard/frontend
    rm -rf node_modules package-lock.json
    npm install --legacy-peer-deps
    npm run build
    deactivate
    # 再重新编译
    cd /workspaces/ceph-fork/build
    ninja -j20


最后是部署
以我的测试环境为例，编译环境为服务器，将所有deb包上传给集群内网环境中的Windows电脑
scp /workspaces/*.deb host@10.10.185.60:"D:/ceph-debs/"

Windows电脑再上传给内网中三台linux虚拟机
$vms = @("192.168.100.11", "192.168.100.12", "192.168.100.13")
foreach ($vm in $vms) {
    Write-Host "正在上传到 $vm ..."
    scp D:\ceph-debs\*.deb ceph@${vm}:/tmp/ceph-debs/
}

虚拟机内部：
    cd /tmp/ceph-debs

    # 安装所有包
    apt install -y ./*.deb

    # 如果有依赖缺失
    apt --fix-broken install -y

    # 验证安装成功
    ceph --version
请务必确保linux虚拟机系统版本也为Ubuntu 20.04，保持版本一致

Boost 安装下载太慢了
    cd ceph-fork.src && wget --progress=bar:force https://mirrors.aliyun.com/blfs/conglomeration/boost/boost_1_75_0.tar.bz2
    tar -xjf boost_1_75_0.tar.bz2
    mv boost_1_75_0 boost
    # 验证
    ls boost/bootstrap.sh

直接dpkg可能还是会报错
    apt-get install -y \
    build-essential cmake git pkg-config libboost-all-dev libssl-dev \
    libedit-dev libxml2-dev libfuse-dev libaio-dev libsnappy-dev \
    liblz4-dev libzstd-dev libkeyutils-dev libudev-dev \
    libcurl4-openssl-dev libfcgi-dev libsqlite3-dev \
    libgoogle-perftools-dev libnss3-dev libbabeltrace-dev \
    libbabeltrace-ctf-dev libpmem-dev libpmemblk-dev \
    debhelper python3-dev python3-pip python3-setuptools \
    python3-venv cython3