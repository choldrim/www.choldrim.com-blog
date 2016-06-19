title: docker在deepin的安装
date: 2016-04-10
tags: 
    - deepin
    - docker
category: deepin

thumbnail: http://7xrkyd.com1.z0.glb.clouddn.com/install-docker-on-deepin/deepin-docker.jpg

---

deepin官方仓库中docker的版本有些旧（我写这篇博客的时候大概是1.5），因此我们选择直接使用docker提供的软件仓库进行安装

其实安装过程很简单，只要选着对应的源即可，deepin15是基于debian sid的，所以选择的源是：
```shell
deb https://apt.dockerproject.org/repo debian-stretch main
```

#### step 1: 添加GPG key
```shell
sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
```

#### step 2: 添加sources
```shell
echo deb https://apt.dockerproject.org/repo debian-stretch main | sudo tee -a /etc/apt/sources.list
```

#### step 3: 更新软件列表
```shell
sudo apt-get update
```

#### step 4: 安装
```shell
sudo apt-get install docker-engine
```

#### step 5: 添加当前用户到docker组
其实经过以上4步就把docker安装上了，但是每次执行docker命令的时候需要加sudo去执行，有些麻烦，只要把当前用户添加docker组就可以免去sudo了
```shell
# 假如你的用户名叫choldrim，则：
sudo usermod -a -G docker choldrim  # 改完后需要重新登录
```

执行命令，测试一下安装成功没 ;)
```shell
docker version
```


ps: 以上教程是在deepin15上进行安装的，deepin14是基于ubuntu Trusty的，把上面的源替换成：
```shell
deb https://apt.dockerproject.org/repo ubuntu-trusty main
```
即可
