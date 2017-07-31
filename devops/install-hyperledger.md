---
layout: page
title: "安装部署"
description: ""
group: navigation
---
{% include JB/setup %}

## 安装部署 ##
Fabric的运行环境是：Host->VM->Docker，所以可以有如下几种不同的安装方式：

### Vagrant安装 ###
这种方式主要是用于搭建开发测试环境，能够在Windows/Mac/Linux等不同的环境下配置出相同的环境来。

#### 准备工作 ####

* 配置环境变量 

请自行替换下面GOPATH的设置：
```
claritys-MBP:~ clarity$ cat ~/.profile
export GOROOT=/usr/local/go
export GOPATH=/Users/clarity/Projects
export PATH=$PATH:/usr/local/sbin
```

* 安装Git并下载源码

```
mkdir -p $GOPATH/src/github.com/hyperledger
cd $GOPATH/src/github.com/hyperledger
git clone http://gerrit.hyperledger.org/r/fabric
```

* 下载并安装Vagrant和VirtualBox

注意：Virtualbox不能下载5.1.X的版本，否则会和Vagrant 1.8.4冲突，出现错误：
```
No usable default provider could be found for your system.
```

#### 运行Vagrant ####
修改ubuntu源：$GOPATH/src/github.com/hyperledger/fabric/scripts/provision/common.sh，在apt-get update前增加一行：
```
sed -i "s/us.archive.ubuntu.com/cn.archive.ubuntu.com/g" /etc/apt/sources.list
```

```
cd $GOPATH/src/github.com/hyperledger/fabric/devenv
vagrant up
```
运行前请配置好翻墙的网络环境，否则某些步骤可能就卡死了。根据网络状况不同，上面的过程可能会很慢，1-5个小时不等。

Proxifier需要设置代理的域名：
```
*.googlesource.com;*.amazonaws.com;security.ubuntu.com;
```

Proxifier不需要DNS解析的域名：
```
*.archive.ubuntu.com;*.gradle.org;*.launchpad.net;github-cloud.s3.amazonaws.com;security.ubuntu.com;
```

如果遇到如下错误：
```
SSL read: error:00000000:lib(0):func(0):reason(0), errno 60
```

可以手动下载解决：
```
vagrant box add hyperledger/fabric-baseimage -c
```

如果遇到如下错误：
```
W: Failed to fetch http://security.ubuntu.com/ubuntu/dists/trusty-security/restricted/source/Sources  Hash Sum mismatch
```

产生的原因是网络中有缓存，源的更新速度不匹配导致的，所以可能换一个网络环境就没有这个问题。我是手动设置Proxifier的规则，让security.ubuntu.com走代理绕过去的。另外，默认的源是us.archive.ubuntu.com，可以先vagrant ssh进入系统，手动设置源为cn.archive.ubuntu.com或者mirrors.163.com，速度会快一点。

如果需要如下错误：
```
Could not get lock /var/lib/apt/lists/lock - open (11: Resource temporarily unavailable)
```

说明是之前异常退出，后台进程还在运行冲突导致的，vagrant ssh进去干掉apt进程，再vagrant up就可以了。

#### 进入Vagrant VM ####
```
vagrant ssh
```

#### 安装docker-compose ####

保存如下内容到install_dockercompose.sh：
```
#!/bin/sh

# docker-compose
curl -L https://github.com/docker/compose/releases/download/1.7.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# write docker command
## display docker process
cat > /bin/dp <<END
#!/bin/sh

# display docker process
docker ps -a
END
chmod +x /bin/dp

## rm container with Docker Container ID/Names
cat > /bin/dr <<END
#!/bin/sh

## container with Docker Container ID/Names
if [ \$# -lt 1 ]; then
    echo "\$0 id/name"
    echo "rm container with Docker Container ID/Names"
    exit 1
fi
id=\$1
containerid=\$(docker ps -a|awk '{if ((\$1~/'"\$id"'/ || \$NF~/'"\$id"'/) && NR >1) {print \$1,\$NF;}}')
echo "stop/rm containerid:" \$containerid
docker stop \$containerid
docker rm \$containerid
END
chmod +x /bin/dr

## docker images
cat > /bin/dm <<END
#!/bin/sh

docker images
END
chmod +x /bin/dm

## docker rm images
cat > /bin/dri <<END
#!/bin/sh

## container with Docker Image ID/Names
if [ \$# -lt 1 ]; then
    image_id=\$(docker images |awk '{if (\$1~/<none>/) {print \$3;}}')
    docker rmi -f \$image_id
    exit 0
fi
id=\$1
image_id=\$(docker images |awk '{if (\$1~/<none>/ || \$1~/'"$id"'/ || \$3~/'"\$id"'/) {print \$3;}}')
docker rmi -f \$image_id
END
chmod +x /bin/dri

## docker exec
cat > /bin/de <<END
#!/bin/sh

if [ \$# -lt 1 ] ; then
	echo "usage: de docker_id"
	exit 0
fi
# exec into
docker exec -it \$1 /bin/bash
END
chmod +x /bin/de

## docker logs
cat > /bin/dl <<END
#!/bin/sh

if [ \$# -lt 1 ] ; then
	echo "usage: dl docker_id"
	exit 0
fi
# logs
docker logs -f \$1
END
chmod +x /bin/dl
```

然后运行脚本：
```
chmod +x install_dockercompose.sh
./install_dockercompose.sh
```

#### 运行单节点网络 ####
编辑一个docker-compose.yml文件：
```
# membersrvc
membersrvc:
  image: hyperledger/fabric-membersrvc
  ports:
    - "50051:50051"
  command: membersrvc

# validating peers
# running with the CA
vp0:
  image: hyperledger/fabric-peer
  ports:
    - "5000:5000"
    - "30303:30303"
    - "30304:30304"
  environment:
    - CORE_PEER_ADDRESSAUTODETECT=true
    - CORE_VM_ENDPOINT=unix:///var/run/docker.sock
    - CORE_LOGGING_LEVEL=DEBUG
    - CORE_PEER_ID=vp0
    - CORE_SECURITY_ENROLLID=test_vp0
    - CORE_SECURITY_ENROLLSECRET=MwYpmSRjupbT
  links:
    - membersrvc
  command: sh -c "sleep 5; peer node start --peer-chaincodedev"
```
然后运行docker-compose up就起来了，可以通过peer命令执行chaincode的deploy/invoke等操作。

### 镜像启动运行 ###
Vagrant启动VM的方式自动会安装Docker和编译Docker镜像文件，然后再运行Fabric。如果有Docker的运行环境，就可以直接同docker-compose一样启动服务了。最简单的方式就是：

#### 安装Docker ####
各种环境的安装参考：[Docker安装](https://docs.docker.com)，[安装docker-compose]。

#### 下载Docker镜像 ####
```
docker pull hyperledger/fabric-membersrvc
docker pull hyperledger/fabric-peer
```

#### 启动Fabric服务 ####
参考：[运行单节点网络]。

### 源码编译安装 ###
源码的安装以Ubuntu/Deian操作系统为例。

#### 准备工作 ####

* 安装Go

[Go安装包下载](https://golang.org/dl)，这个应该也是要翻墙下的。

* 安装Docker

各种环境的安装参考：[Docker安装](https://docs.docker.com)，[安装docker-compose]。

如果要非root环境执行docker命令，可以把用户xxx添加到docker组：
```
usermod -a -G docker xxx
```

* 配置环境变量

请自行替换下面GOPATH的设置：
```
claritys-MBP:~ clarity$ cat ~/.profile
export GOROOT=/usr/local/go
export GOPATH=/Users/clarity/Projects
export PATH=$PATH:/usr/local/sbin
```

* 安装Git并下载源码

```
mkdir -p $GOPATH/src/github.com/hyperledger
cd $GOPATH/src/github.com/hyperledger
git clone https://github.com/hyperledger/fabric.git
```

* 安装RocksDB

```
apt-get install -y libsnappy-dev zlib1g-dev libbz2-dev
cd /tmp
git clone https://github.com/facebook/rocksdb.git
cd rocksdb
git checkout v4.1
PORTABLE=1 make shared_lib
INSTALL_PATH=/usr/local make install-shared
```

* 安装pip

```
apt-get install -y python-pip
pip install --upgrade pip
pip install behave nose docker-compose
pip install -I flask#####0.10.1 python-dateutil#####2.2 pytz#####2014.3 pyyaml#####3.10 couchdb#####1.0 flask-cors#####2.0.1 requests#####2.4.3
```

* 错误：Release: The following signatures couldn't be verified because the public key is not available: 
 NO_PUBKEY 2EA8F35793D8809A

```
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 2EA8F35793D8809
```

* 错误：peer: error while loading shared libraries: librocksdb.so.4.1: cannot open shared object file: No such file or directory

```
cp /usr/local/lib/librocksdb.so.4.1* /usr/lib/
```
也可以把/usr/local/lib目录增加到/etc/ld.so.conf目录下。或者编译的时候改成：INSTALL_PATH=/usr make install-shared

#### 编译Docker镜像 ####
```
cd $GOPATH/src/github.com/hyperledger/fabric
make membersrvc-image
make peer-image
```

#### 运行Fabric服务 ####

参考：[运行单节点网络]。
