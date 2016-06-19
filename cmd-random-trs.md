title: 利用urandom获取随机字符串
date: 2016-04-17
tags: [shell,linux-command,random]
category: linux-cmdline
thumbnail: http://7xrkyd.com1.z0.glb.clouddn.com/shell-random-str/random-str.jpg

---

## urandom
先贴上命令行：
```shell
# 生成一个32位长度的随机字符串
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1
```

用好几个管道去一个字符串不是为了卖弄命令行的语法糖，只是为了在不需要安装其他依赖的情况下也能简单的得到随机字符串(cat/tr/fold/head 都是在coreutils这个包里的)

假如你在写shell脚本需要生成随机字符串的话可以考虑一下使用这个，而不是直接安装一个新的依赖

也可以把上面那条命令写成一个shell脚本
```shell
#!/bin/bash -e

count=$1

echo $(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w $count | head -n 1)
```

放在你PATH所包含的路径即可当成简单的字符串生成器使用了

## pwgen
另外，假如需要用到可定制的、更高级的随机字符串器，可以使用pwgen这个软件
```shell
# 生成一个32位无数字的随机字符串
pwgen -n 32 -1 -0
# >> johceequalaexohviGhahfefohphooja
```
