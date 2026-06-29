此版本增加了.devcontainer配置脚本，推荐通过VS Code的Dev Containers插件使用

构建版本为：Ubuntu 20.04
Ceph Tag: v17.2.7-custom
Git Branch: cache

主要针对BlueStore层的Onode Cache的缓存替换算法做了S3FIFO的修改与适配
同时增加了客户端层面关于文件更细粒度的缓存命中统计

apt install tmux -y
tmux new -s ceph-build
tmux attach -t ceph-build
git config --global url."git@github.com:".insteadOf "https://github.com/"

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
Node1:
    # 安装工具
    apt install -y dpkg-dev apt-utils nginx
    # 创建仓库目录
    mkdir -p /var/www/html/ceph-repo
    # 复制 deb 包
    cp /tmp/ceph-debs/*.deb /var/www/html/ceph-repo/
    # 生成仓库索引
    cd /var/www/html/ceph-repo
    dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz
    dpkg-scanpackages . > Packages

    cat > /etc/nginx/sites-available/ceph-repo <<'EOF'
    server {
        listen 8080;
        root /var/www/html;
        autoindex on;
    }
    EOF

    ln -s /etc/nginx/sites-available/ceph-repo /etc/nginx/sites-enabled/
    nginx -t && systemctl restart nginx

    # 在所有节点执行
    echo "deb [trusted=yes] http://192.168.100.11:8080/ceph-repo ./" \
    > /etc/apt/sources.list.d/ceph-local.list

    # 查找官方 ceph 源文件
    ls /etc/apt/sources.list.d/ | grep ceph

    # 禁用它（假设文件名为 ceph.list）
    mv /etc/apt/sources.list.d/ceph.list /etc/apt/sources.list.d/ceph.list.bak

    # 同样禁用 chacra 源
    ls /etc/apt/sources.list.d/ | grep chacra
    mv /etc/apt/sources.list.d/chacra*.list /tmp/

    apt update
    
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