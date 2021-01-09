---
title: Docker 部署 hexo
tag: 教程
---

# Docker 部署 hexo

## 方法一、使用 spurin/hexo 架设 hexo

spurin/hexo 是至2020年中保持更新的一个 hexo 的 docker 镜像，一键部署，比较方便

```
docker create --name=hexo \
-e HEXO_SERVER_PORT=4000 \
-e GIT_USER="HEHEKING" \
-e GIT_EMAIL="2351965997@qq.com" \
-v /root/blog/otaku-home:/app \
-p 4000:4000 \
spurin/hexo
```

详细步骤可参考https://www.dazhuanlan.com/2019/12/30/5e09d1bfb22d7/

## 方法二、使用 zeusro/hexo 架设 hexo

zeusro/hexo 是不再维护的 hexo 一键部署镜像，因为没有必要。他部署 hexo 只需要一个 **公开 PUBLIC** 的 github 连接。

```
docker run --name hexo -d -p 4000:4000 zeusro/hexo:latest  --env PUBLIC_HEXO_GITHUB_URL=https://github.com/HEHEKING/otaku-home.git 
```



## 方法三、从 Centos7 镜像从零部署

这个办法我估摸着就和 hexo 部署教程**一模一样**了几乎。好像没有使用 docker 的必要？

但 docker 对于一部分强迫症患者症状减轻真的很有效。

### 1. 初始化容器

```shell
# run 一个 name 为 hexo 的 centos7，并挂载4000端口，同时也可以使用 -v 挂载一下之后会使用到的目录
docker run  --name hexo -it -p 4000:4000 -v /root/hexo:/root/otaku-home centos:7 /bin/bash
# 在之后可以使用这个命令回到该容器
docker exec -it hexo /bin/bash
# 安装基础库
yum -y install git gcc gcc-c++ curl wget
```

**PS**：对 docker 还不够熟悉的话，可以先看完整篇文章，再一步步动手操作，这里的挂载目录部分可能会引发**误解**

### 2. 安装nodejs14

对于 nodejs 来说，版本自然是越高越好。

```shell
cd /usr/local
# 创建安装目录
mkdir nodejs
cd nodejs
# 下载node
wget https://npm.taobao.org/mirrors/node/latest-v14.x/node-v14.2.0-linux-x64.tar.gz
# 解压缩
tar zxvf node-v14.2.0-linux-x64.tar.gz
# 创建快捷方式
ln -s /usr/local/nodejs/node-v14.2.0-linux-x64/bin/npm /usr/local/bin/
ln -s /usr/local/nodejs/node-v14.2.0-linux-x64/bin/node /usr/local/bin/
# 检测 node 和 npm 的版本
node -v
v14.2.0
npm -v
6.14.4
```

值得一提的是，这种安装方式，如果没有配置 /etc/profile，全局安装是失效的，事实上局部安装也是无法访问的，因此我建议每建立一个 npm 项目，都**更新**一次环境变量

### 3. npm 单次换源

换个淘宝源，防止之后使用 npm 时进行不必要的等待，这一行为是仅作用于这一次会话的。

```
npm install --registry=https://registry.npm.taobao.org
```

**PS**：`pwd` 命令可以帮助我们获取当前路径，这在接下来的步骤中可能有所帮助

### 4. 安装 hexo（初次接触 hexo）

如果我们是**初次建立**hexo blog，应该是没有 hexo 代码仓库的，那么这里可以直接如官方文档操作。如果已经有 hexo.git 的可以直接跳到**第5步**

**PS**：这里 hexo 的实际部署位置为：/root/npm/hexo，那么之前初始化 docker 挂载的时候，也应该是使用这个目录。如：**-v hexo:/root/hexo:/root/npm/hexo**

```shell
cd /root/npm
# 本地安装 hexo
npm install hexo
# 安装 hexo 服务器
npm install hexo-server
# 这里使用 echo 命令，将该目录下的 npm 库 加入环境变量
echo 'export PATH=$PATH:/root/npm/node_modules/.bin' >> /etc/profile
# 或者我们也可以使用 vi 直接编辑它
vi /etc/profile
# 更新环境变量
source /etc/profile
```

到这一步，我们就可以开始架设 hexo 了

```shell
# 建立 hexo
mkdir hexo
# hexo 初始化,注意在进行 hexo 初始化的时候，必须有目标文件夹，且目标文件夹必须为空。
hexo init ./hexo
```

这时，它会从官方的代码仓库克隆一份原始模板。

```shell
# 编译 hexo
hexo g
# 启动 hexo 服务器
hexo server
```

至此，一个 hexo 服务器已被架设在本机的 4000 端口。

### 5. 通过 git 部署 hexo

在第一步中，我们 -v 挂载的目录，在这里就是 hexo 的实际目录

**PS**：在这里，我们需要注意，github的代码仓库，clone下来时，是一个目录，如：**HEHEKING/otaku-home** 仓库，使用 git clone 后会在该目录创建一个 otaku-home 目录

```shell
# 进入目录
cd /root

# 克隆仓库，如果该仓库是私密的，需要输入 github 的账户密码进行验证。
git clone https://github.com/HEHEKING/otaku-home.git
# 进入目录
cd ./otaku-home
# npm 依赖更新，npm 会下载项目中缺失的依赖
npm install
# 这时，我们依然需要更新一下环境变量，但是要注意目录
echo 'export PATH=$PATH:/root/otaku-home/node_modules/.bin' >> /etc/profile
# 更新环境变量
source /etc/profile

# 开始部署
hexo g
hexo server
```

至此，将 git 上的 hexo 仓库部署到 docker 已完毕。

这里，我们使用 ctrl + P + Q 可以退出当前 docker 容器，回到主容器

当我们 docker attach hexo 时，会发现看似卡了，实际上没有问题，因为 hexo server 不再打印日志了，ctrl + C 退出即可。

**PS**：配合 docker commit 保存容器为镜像会方便日后使用

## 总结

前两种方式的镜像资源会小很多。

我使用了第三种。直接 git 部署，还没测试过 hexo init 那块教程

如果有问题，亲自调试吧_(:з」∠)_
