title: 在nginx上使用自签名SSL证书
date: 2016-06-10
tags: [nginx, ssl, certificate, CA]
category: nginx

---

![nginx ssl](http://7xrkyd.com1.z0.glb.clouddn.com/nginx-ssl-with-self-signed-cert/super-cacher-with-ssl-and-ngimx.jpg)

## step 1: 创建Server Key 和 Certificate Signing Request

- 生成server private key

 ```shell
 # 2048 是秘钥的长度
 openssl genrsa -out ssl.key 2048
 ```

- 使用private key创建CSR (Certificate Signing Request)

 ```shell
 openssl req -new -key ssl.key -out ssl.csr
 ```
 
 ```shell
 输出会询问关于这个证书的信息
 You are about to be asked to enter information that will be incorporated
 into your certificate request.
 What you are about to enter is what is called a Distinguished Name or a DN.
 There are quite a few fields but you can leave some blank
 For some fields there will be a default value,
 If you enter '.', the field will be left blank.
 -----
 Country Name (2 letter code) [AU]:
 State or Province Name (full name) [Some-State]:
 Locality Name (eg, city) []:
 Organization Name (eg, company) [Internet Widgits Pty Ltd]:
 Organizational Unit Name (eg, section) []:
 Common Name (e.g. server FQDN or YOUR name) []:www.test.com  # 必填
 Email Address []:         

 Please enter the following 'extra' attributes
 to be sent with your certificate request
 A challenge password []:
 An optional company name []:
```

**注意：**

其他都可以不填，但`Common Name (e.g. server FQDN or YOUR name) []:` 一定要填，填写你需要使用这个证书提供服务的域名(不打算使用域名的就填IP)


## step 2: SSL证书签发

```shell
# -days 365是签发的证书过期时间
openssl x509 -req -days 365 -in ssl.csr -signkey ssl.key -out ssl.crt
```

## step 3: 配置nginx

- 拷贝证书和私钥到`nginx`所在目录
 (这步可以省略，后面的nginx配置中`ssl_certificate`和`ssl_certificate_key`位置对应即可)

 ```shell
sudo mkdir -p /etc/nginx/ssl
sudo cp ssl.key /etc/nginx/ssl/www.test.com.key
sudo cp ssl.crt /etc/nginx/ssl/www.test.com.crt
```

- 放一个简单的页面到`/var/www/test`

 ```shell
sudo mkdir -p /var/www/test
echo '<h1>hello nginx</h1>' | sudo tee -a /var/www/test/index.html
```

- 编写nginx配置文件
 ```shell
sudo vim /etc/nginx/sites-enabled/www.test.com.conf
```

 添加以下内容
 ```conf
server{
    listen 443;
    server_name www.test.com;

    ssl on;
    ssl_certificate /etc/nginx/ssl/www.test.com.crt;
    ssl_certificate_key /etc/nginx/ssl/www.test.com.key;

    root /var/www/test;
}
```

 保存后记得reload一下nginx

 ```shell
sudo nginx -s reload
```

## 验证

- 因为我例子中签的证书CN是www.test.com，验证前先在本地`/etc/hosts`文件添加

 ```conf
127.0.0.1 www.test.com
```

- 验证方式有很多种，这里提供两种比较常用的

 1. curl验证
   直接curl加地址
   ```shell
   curl https://www.test.com
   #
   #curl: (60) server certificate verification failed. CAfile: /etc/ssl/certs/ca-certificates.crt CRLfile: none
   #More details here: http://curl.haxx.se/docs/sslcerts.html

   #curl performs SSL certificate verification by default, using a "bundle"
   # of Certificate Authority (CA) public keys (CA certs). If the default
   # bundle file isn't adequate, you can specify an alternate file
   # using the --cacert option.
   #If this HTTPS server uses a certificate signed by a CA represented in
   # the bundle, the certificate verification probably failed due to a
   # problem with the certificate (it might be expired, or the name might
   # not match the domain name in the URL).
   #If you'd like to turn off curl's verification of the certificate, use
   # the -k (or --insecure) option.
   ```

   报服务器证书验证失败错误

   加上`--cacert`
   ```shell
   curl https://www.test.com --cacert ssl.crt   
   #
   # <h1>hello nginx</h1>
   ```

   验证通过 :D

  2. 浏览器验证 (我用的是chrome)

   浏览器直接打开`https://www.test.com`，发现chrome提示`您的链接不是私密链接`
   ![警告](http://7xrkyd.com1.z0.glb.clouddn.com/nginx-ssl-with-self-signed-cert/%E6%B7%B1%E5%BA%A6%E6%88%AA%E5%9B%BE20160610222433.png)

   是因为chrome默认自带的证书库中没找到对应的证书，无法加密发送数据。

   查看服务器证书信息
   ![](http://7xrkyd.com1.z0.glb.clouddn.com/nginx-ssl-with-self-signed-cert/%E6%B7%B1%E5%BA%A6%E6%88%AA%E5%9B%BE20160610222527.png)

   可以看到加载的服务器证书CN和有效期都是对的
   ![](http://7xrkyd.com1.z0.glb.clouddn.com/nginx-ssl-with-self-signed-cert/%E6%B7%B1%E5%BA%A6%E6%88%AA%E5%9B%BE20160610222611.png)

   手动导入证书
   浏览器打开证书配置页面[chrome://settings/certificates](chrome://settings/certificates)
   ![导入](http://7xrkyd.com1.z0.glb.clouddn.com/nginx-ssl-with-self-signed-cert/%E6%B7%B1%E5%BA%A6%E6%88%AA%E5%9B%BE20160610223903.png)

   点击导入，选中`ssl.crt`文件，勾选`信任该证书，以标识网站的身份。`，确定。
   ![](http://7xrkyd.com1.z0.glb.clouddn.com/nginx-ssl-with-self-signed-cert/%E6%B7%B1%E5%BA%A6%E6%88%AA%E5%9B%BE20160610224429.png)

   刷新`https://www.test.com/`页面，可以看到https小锁头变绿了，验证通过 :D
   ![](http://7xrkyd.com1.z0.glb.clouddn.com/nginx-ssl-with-self-signed-cert/%E6%B7%B1%E5%BA%A6%E6%88%AA%E5%9B%BE20160610233043.png)


对CA证书的签发，我也是在学习中，讲得比较肤浅，暂且先在此处抛个砖吧。XD

(end)
