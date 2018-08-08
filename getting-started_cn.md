# 入门

## 安装

您可以使用此仓库 [https://packagecloud.io/Lazin/Akumuli](https://packagecloud.io/Lazin/Akumuli) 安装Akumuli，它包含下列操作系统所需的软件包：
* Ubuntu 14.04 \(amd64\)
* Ubuntu 16.04 \(amd64 and arm64\)
* Debian Stretch \(amd64\)
* Debian Jessie \(amd64\)
* Red Hat Enterprise Linux 7 \(amd64\)
* CentOS 7 \(amd64\)

支持OS X，但目前没有提供软件包。

此外，你还可以使用Docker仓库 [Docker repository](https://hub.docker.com/r/akumuli/akumuli/)。

### 由源代码构建

#### Ubuntu / Debian

**必备条件**

**自动**

* 运行`prerequisites.sh`，它将尝试使用最优选项。

**手动**

如果自动化脚本无法工作：

* Boost:

  `sudo apt-get install libboost-all-dev`

* log4cxx:

   `sudo apt-get install log4cxx`或者在Ubuntu 16.04之上使用`sudo apt-get install liblog4cxx-dev`

* jemalloc:

  `sudo apt-get install libjemalloc-dev`

* microhttpd:

  `sudo apt-get install libmicrohttpd-dev`

* APR:

  `sudo apt-get install libapr1-dev libaprutil1-dev libaprutil1-dbd-sqlite3`

* SQLite:

  `sudo apt-get install libsqlite3-dev`

* Cmake:

  `sudo apt-get install cmake`

**构建**

1. `cmake .`
2. `make -j4`

#### Centos 7 / RHEL7 / Fedora

**自动**

* 运行`prerequisites.sh`，它将尝试使用最优选项。

**手动**

如果自动化脚本无法工作：

* Boost:

  `sudo yum install boost boost-devel`

* log4cxx:

  `sudo yum install log4cxx log4cxx-devel`

* jemalloc:

  `sudo yum install jemalloc-devel`

* microhttpd:

  `sudo yum install libmicrohttpd-devel`

* APR:

  `sudo yum install apr-devel apr-util-devel apr-util-sqlite`

* SQLite

  `sudo yum install sqlite sqlite-devel`

* Cmake:

  `sudo yum install cmake`

**构建**

1. `cmake .`
2. `make -j4`
3. `make`

## 初始步骤

### 配置

首先创建配置文件，可以使用如下命令完成：

```text
> akumulid --init
OK configuration file created at: "/home/username/.akumulid"
```

现在，我们可以编辑配置文件 `~/.akumulid`，它包含默认配置和注释。两个主要的配置参数是 `path` 和 `nvolumes`，前者应该指向存储数据库文件的目录，akumuli默认将文件保存在 `~/.akumuli` 目录，可以变更为任何想要存放的位置（大多数时候个人倾向于使用 `path=/tmp` 用于测试运行）；后者 `nvolumes` 参数是用来存放数据的卷数量，每个卷尺寸是4Gb，要有预见性的选择这个值。

### 数据库创建

接下来我们使用如下命令来创建数据库本身：

```text
> akumulid --create
OK database created, path: /home/username/.akumuli
```

此时可以检查数据库文件是不是真的已经创建于`~/.akumuli`，这个目录不应该为空。（注意：可以通过运行命令`akumulid --delete`删除所有文件。）

### 变更Akumuli配置

返回配置文件\(`~/.akumulid`\)，阅读配置文件中的参数描述，最重要的参数包括：

* `path` - 告诉Akumuli应该在何处存储数据库卷\(默认值是 ~/.akumuli\)
* `nvolumes` - 应该创建的卷数\(这个参数仅仅在运行`akumulid --create`命令时使用\)，如果`nvolumes`设置为0，将会在不删除旧数据的前提下按需扩展存储。
* `volume_size` - 单个卷的尺寸\(这个参数仅仅在运行`akumulid --create`命令时使用\)
* `HTTP.port` - HTTP服务使用的端口号
* `TCP.port` - TCP服务使用的端口号
* `TCP.pool_size` - 用于数据处理的线程数\(应该小于CPU核数，如果设置为0系统将在启动时尝试选择最优数量\)
* `UDP.port` - UDP服务使用的端口号
* `UDP.pool_size` - 用于数据处理的线程数\(应该小于CPU核数\)
* Log4cpp配置

### 启动服务器

作为服务器运行`akumulid`，不要指定任何参数：

```text
> akumulid
OK UDP  server started, port: 8383
OK TCP  server started, port: 8282
OK HTTP server started, port: 8181
```

至此，已经可以通过TCP或者UDP写入数据，或者使用HTTP读取数据。



### 原文以外的补充

1、RHEL7 光盘缺少 log4cxx-devel、libmicrohttpd-devel，导致cmake步骤报错，使用如下命令安装

yum install http://mirror.centos.org/centos/7/os/x86_64/Packages/log4cxx-devel-0.10.0-16.el7.x86_64.rpm
yum install http://mirror.centos.org/centos/7/os/x86_64/Packages/libmicrohttpd-devel-0.9.33-2.el7.x86_64.rpm

网页

https://centos.pkgs.org/7/centos-x86_64/log4cxx-devel-0.10.0-16.el7.x86_64.rpm.html
https://centos.pkgs.org/7/centos-x86_64/libmicrohttpd-devel-0.9.33-2.el7.x86_64.rpm.html

2、缺少apr-util-sqlite

RHEL7下 `akumulid --create` 可能会遇到Aborted (core dumped)，原因是没有安装apr-util-sqlite

yum install http://mirror.centos.org/centos/7/os/x86_64/Packages/apr-util-sqlite-1.5.2-6.el7.x86_64.rpm

网页

https://centos.pkgs.org/7/centos-x86_64/apr-util-sqlite-1.5.2-6.el7.x86_64.rpm.html

3、启动时TCP有两个端口，与文档所述不同

```text
OK HTTP server started, port: 8181
OK TCP server started, RESP port: 8282, OpenTSDB port: 4242
OK UDP server started, port: 8383
```



