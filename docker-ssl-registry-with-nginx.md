title: 借助nginx搭建docker SSL registry
date: 2016-06-11
tags: [docker, docker-registry, ssl, nginx]
category: docker

---

![docker registry](http://7xrkyd.com1.z0.glb.clouddn.com/docker-ssl-registry-with-nginx/screen-shot-2014-12-05-at-3-37-43-pm.png)

搭建docker SSL验证仓库方式不止一种，但是不管你用哪种方式搭建，都必须要有证书和私钥

证书的生成可以按照我的前一篇博客[在nginx上使用自签名SSL证书](http://www.choldrim.com/2016/06/09/nginx-ssl-with-self-signed-cert/)所说的方式自签名一个证书；也可以自己去购买x一个公认的证书，但是这个好贵，一般公司才会选择这种方案，如果你是RMB玩家，推荐这种，省事；如果想让你的证书是公认的，又想不花钱，可以试试[let's encrypt](https://letsencrypt.org/)。

好，下面就讲一下，拿到（自签或者公认）证书后怎么搭建一个SSL验证的docker仓库

假设我们的仓库域名是：`www.test.com`

如果本地网络没有做好dns，则在本地`/etc/hosts`文件添加
```conf
127.0.0.1 www.test.com
```

## step 1: 启动一个http的docker registry

```shell
docker run --rm --name registry -p 5000:5000 -v /tmp/docker-registry:/var/lib/registry registry:2.4.0
```

测试：测试方式是直接访问`/v2/`，返回`{}`表示成功
    
```shell
curl http://127.0.0.1:5000/v2/

# {}%
```


## step 2: 配置nginx

假设证书和秘钥的名字分别为：`www.test.com.crt`, `www.test.com.key`

都放在`/etc/nginx/ssl`目录下

```shell
sudo vim /etc/nginx/sites-enabled/www.test.com.conf
```

```conf
server{
    listen 443;
    server_name www.test.com;

    ssl on;
    ssl_certificate /etc/nginx/ssl/www.test.com.crt;
    ssl_certificate_key /etc/nginx/ssl/www.test.com.key;

    location / {
        proxy_pass http://127.0.0.1:5000; # 方向代理到本地的registry
    }
}
```

配置完后reload一下

```shell
sudo nginx -s reload
```

**测试一下**
```shell
# www.test.com.crt 就是和上面nginx使用的是同一个
curl https://www.test.com/v2/ --cacert www.test.com.crt

# {}%
```

没报证书验证失败错误，返回`{}`则表示通过

## step 3: 分发证书到docker客户端

(使用公认证书的略过这一步)

按照docker的约定，自签证书需要把证书放在`/etc/docker/certs.d`

在`/etc/docker/certs.d`下创建目录`www.test.com`，目录名字也是有规定的，必须为`$IP:$PORT`格式，因为https默认就是443，我们仓库服务端打开的也是443端口，因此`PORT`可以缺省不写

```shell
# 创建证书存放目录
sudo mkdir -pv /etc/docker/certs.d/www.test.com

# 拷贝证书到docker约定的目录下
sudo cp www.test.com.crt /etc/docker/certs.d/www.test.com/ca.crt
```

## 测试

此时的仓库服务端没有镜像，先做一个`push`动作，先从docker hub上拉取一下小镜像回来，用于测试

```shell
docker pull busybox
docker tag busybox www.test.com/busybox
```

测试`push`推送镜像

```shell
docker push www.test.com/busybox
#
# The push refers to a repository [www.test.com/busybox]
# 5f70bf18a086: Pushed 
# 1834950e52ce: Pushed 
# latest: digest: sha256:9e19eb0c215b760dc7f268402ffe83cb0f063bbcbb181bc25d2d75a9531ea117 size: 733
#
```

推送成功

测试pull
```shell
# 先删除原本的www.test.com/busybox镜像
docker rmi www.test.com/busybox
 
# Untagged: www.test.com/busybox:latest

docker pull www.test.com/busybox
 
# Using default tag: latest
# latest: Pulling from busybox
#
# Digest: sha256:9e19eb0c215b760dc7f268402ffe83cb0f063bbcbb181bc25d2d75a9531ea117
# Status: Downloaded newer image for www.test.com/busybox:latest
```

测试通过

## 用户名验证
如果要加入账户验证，则在nginx配置中，增加

```conf
location / {
    proxy_pass http://127.0.0.1:5000;
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

生成`.htpasswd`文件
```shell
sudo htpasswd -c -b /etc/nginx/.htpasswd choldrim 123456
```

修改配置后reload一下

```shell
sudo nginx -s reload
```

```shell
# 直接拉取，发现失败了
docker pull www.test.com/busybox

# Using default tag: latest
# Pulling repository www.test.com/busybox
# unauthorized: authentication required


# 登录
docker login -u choldrim -p 123456 www.test.com

# Login Succeeded


# 再次拉取
docker pull www.test.com/busybox
# Using default tag: latest
# latest: Pulling from busybox
# Digest: 
# sha256:9e19eb0c215b760dc7f268402ffe83cb0f063bbcbb181bc25d2d75a9531ea117
# Status: Downloaded newer image for www.test.com/busybox:latest
```

 (end)