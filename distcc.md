title: distcc分布式编译初探
date: 2016-01-21
tags: distcc
thumbnail: http://7xrkyd.com1.z0.glb.clouddn.com/distcc/blog_ppb_s.jpg
---

公司的CI编译有些慢，前阵子还上了内核，编译一次，1小时+的样子，于是，想想能不能做一些加快的分布式编译的东西，一来节省大家等待CI的时间，二来好好利用服务器资源

想必找到这篇文章的也是对distcc有所了解的了，所以distcc的历史就不多说了

## 环境：
machine：i7 8G、i7 8G、i7 16G
docker image: Debian 8 (jessie)
docker version: 1.9.1

## distcc安装配置
## servers 端 （提供编译服务的节点）
#### 1. 安装distcc
```shell
# 在容器中执行
apt-get install distcc
```

#### 2. 安装编译环境
安装g++即可
```shell
# 容器中执行
apt-get install g++
```

#### 3. 拉起distcc daemon
```shell
distccd --allow 10.0.0.1/16 --daemon --log-file=/var/log/distcc.log

# 10.0.0.1/16 是我们公司内部的网络情况
# 一般不让root用户直接把服务拉起来，创建一个普通用户启动即可

```

## client 端（编译服务使用者）
#### 1. 同样，client端也需要安装distcc
```shell
apt-get install distcc
```

#### 2. 设置hosts
有三种方式：
##### DISTCC_HOSTS 环境变量
**I.** 直接将hosts设置在环境变量中，如：
```shell
export DISTCC_HOSTS='--randomize localhost 10.0.1.62,cpp,lzo
 hostsB,cpp,lzo hostC,cpp,lzo'
# cpp: 对该server启动pump模式
# lzo: 使用LZO压缩算法传输文件
```

**II.** $DISTCC_DIR/hosts
$DISTCC_DIR 默认为家目录下的 `.distcc` 目录

**III.** /etc/distcc/hosts
（在DISTCC_HOSTS环境变量和$HOME/.distcc/hosts都不存在的时候会读取这个文件）
三者的写法都是一样的，具体语法，看后面附。

## 使用：
### 普通模式：
这种方式最简单，直接指定 CC 和 Cxx 环境变量即可
```shell
make CC=distcc CXX=distcc
```

### pump 泵模式：
自distcc版本3.0开始，加入了基于python的新工具pump。其功能是将头文件也随同源码一起发送至编译服务器。将部分预编译工作也进行分布式处理。从而进一步的提升了编译效率。官方给出的数字是对于文件传输、编译过程有10倍的效率提升。
要求：
I. server and client distcc version >= 3.0
II. Another important assumption is that the include configuration of all machines must be identical. Thus the headers under the default system path must be the same on all servers and all clients 
(前半句没懂，后半句是说系统库的路径也需要和编译发起机的系统库路径一致，换句话就是，客户端依赖什么库，服务端就有什么库)

```shell
example:
distcc-pump make CC=distcc CXX=distcc
```

### MASQUERADING 伪装模式：
distcc文档是这么做的：
```shell
# mkdir /usr/lib/distcc/bin
# cd /usr/lib/distcc/bin
# ln -s ../../../bin/distcc gcc
# ln -s ../../../bin/distcc cc
# ln -s ../../../bin/distcc g++
# ln -s ../../../bin/distcc c++
```

官方的做法感觉没那么方便，我是直接这样子：
```shell
export PATH=/usr/lib/distcc:$PATH # 或者是直接加入 .bashrc/.zshrc
make  # 因为gcc/g++等PATH被覆盖，所以直接执行make就可以了
```

### ccache
对中间文件做了缓存，根据预编译结果，通过hash表索引进行缓存，在二次编译时加速编译。Ccache具有以下特性：
 
1.在命中/缺失上是静态的
2.自动的缓存大小管理
3.可以缓存产生的编译警告
4.容易安装
5.很低的开销

```shell
# usage
apt-get install ccache
export CCACHE_PREFIX="distcc"
make CC="ccache gcc"
```
ccache不支持distcc pump模式
(ccache我试用了一下，似乎没感觉到效果，就没使用了，估计是我配置错误吧 :(  )

## debuild 使用distcc
debuild默认的环境变量为：`/usr/sbin:/usr/bin:/sbin:/bin:/usr/bin/X11`，
不被用户的PATH所影响，我用的是masquerading模式，
因此需要把/usr/lib/distcc传入debuild的PATH
debuild 可以 --prepend-path 传入PATH，如下：
```shell
yes | debuild --prepend-path /usr/lib/distcc  -us -uc  -j20
```

## 负载均衡的一些想法
一种方式是使用 dmucs

另一种是自己想的：动态路由
类似伪装模式：
对make做一个包装
- 存储一个监控用的值（可以是上次修改时间，可以是倒数）
- 每次make都会先去检测这个值，判断是否到了拉取distcc-hosts的时候了
- 如果需要则拉取路由，拉取，修改监控指，然后执行make
- 不需要则跳过，继续make

好处是如果某台编译机下架了或者上了新的编译机可以很方便的更新客户端的hosts


### 附1：DISTCC_HOSTS 语法
DISTCC_HOSTS = HOSTSPEC ...
HOSTSPEC = LOCAL_HOST | SSH_HOST | TCP_HOST | OLDSTYLE_TCP_HOST
| GLOBAL_OPTION
| ZEROCONF
LOCAL_HOST = localhost[/LIMIT]
| --localslots=<int>
| --localslots_cpp=<int>
SSH_HOST = [USER]@HOSTID[/LIMIT][:COMMAND][OPTIONS]
TCP_HOST = HOSTID[:PORT][/LIMIT][OPTIONS]
OLDSTYLE_TCP_HOST = HOSTID[/LIMIT][:PORT][OPTIONS]
HOSTID = HOSTNAME | IPV4 | IPV6
OPTIONS = ,OPTION[OPTIONS]
OPTION = lzo | cpp | auth
GLOBAL_OPTION = --randomize
ZEROCONF = +zeroconf

(理解起来也不难，我就不翻译了，加上我英文不好，怕毁了意境 >.<)

Here are some individual examples of the syntax:
* localhost
The literal word "localhost" is interpreted specially to cause compilations to be directly executed, rather than passed to a daemon on the local machine. If you do want to connect to a daemon on the local machine for testing, then give the machine's IP address or real hostname. (This will be slower.)
* IPV6
A literal IPv6 address enclosed in square brackets, such as [::1]
* IPV4
A literal IPv4 address, such as 10.0.0.1
* HOSTNAME
A hostname to be looked up using the resolver.
* :PORT
Connect to a specified decimal port number, rather than the default of 3632.
* @HOSTID
Connect to the host over SSH, rather than TCP. Options for the SSH connection can be set in ~/.ssh/config
* USER@
Connect to the host over SSH as a specified username.
* :COMMAND
Connect over SSH, and use a specified path to find the distccd server. This is normally only needed if for some reason you can't install distccd into a directory on the default PATH for SSH connections. Use this if you get errors like "distccd: command not found" in SSH mode.
* /LIMIT
A decimal limit can be added to any host specification to restrict the number of jobs that this client will send to the machine. The limit defaults to four per host (two for localhost), but may be further restricted by the server. You should only need to increase this for servers with more than two processors.
* ,lzo
Enables LZO compression for this TCP or SSH host.
* ,cpp
Enables distcc-pump mode for this host. Note: the build command must be wrapped in the pump script in order to start the include server.
* ,auth
Enables GSSAPI-based mutual authentication for this host.
* --randomize
Randomize the order of the host list before execution.
* +zeroconf

This option is only available if distcc was compiled with Avahi support enabled at configure time. When this special entry is present in the hosts list, distcc will use Avahi Zeroconf DNS Service Discovery (DNS-SD) to locate any available distccd servers on the local network. This avoids the need to explicitly list the host names or IP addresses of the distcc server machines. The distccd servers must have been started with the "--zeroconf" option to distccd. An important caveat is that in the current implementation, pump mode (",cpp") and compression (",lzo") will never be used for hosts located via zeroconf.
Here is an example demonstrating some possibilities:
```conf
localhost/2 @bigman/16:/opt/bin/distccd oldmachine:4200/1
# #号是注释
distant/3,lzo
```
#### 附2：遇到的一些奇怪的现象 和 解决
1. distcc-pump 遇到的问题
I. 首先在一个依赖只安装了编译环境（g++/gcc）distcc
使用pump：
```shell
# 我此处编译的是 deepin控制中心 master分支的代码
distcc-pump make CC=distcc CXX=distcc
```
会有失败的，如： main.cpp 在remote host编译失败

II. 然后使用伪装模式(PATH=/usr/lib/distcc:$PATH)：
```shell
# 直接编译
make
```
这样子是正常的

II. 最后给server安装了dde-control-center依赖的编译环境，两种模式均可编译通过

A: 后来找文档才知道，pump模式需要本地代码库也路径一致，但是server没安装编译依赖，
而伪装模式（或者普通模式：make CC=distcc CXX=distcc）


2.
debuild 使用distcc，一开始我写的是这样
```shell
yes | debuild -us -uc -j20 --prepend-path /usr/lib/distcc
```
然后就蛋疼了，一直报错，看了文档，确实是支持 --prepend-path 参数的呀，
后来才发现
```shell
debuild [debuild options] [dpkg-buildpackage options] [--lintian-opts lintian
options]
```
这传参设计，晕了，居然所有的options都拼在后边，读取options方式是：
先给debuild，如果debuild出现不是他的参数了，就把后面的参数给dpkg-buildpackage命令，
假如一不小心把debuild的参数加到了dpkg-buildpackage的参数后面就报错了

