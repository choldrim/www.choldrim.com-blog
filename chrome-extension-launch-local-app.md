title: chrome插件调用本地程序
date: 2016-03-06 09:01:37
tags: chrome-extension
category: chrome-extension
thumbnail: http://7xrkyd.com1.z0.glb.clouddn.com/chrome-extension-launch-local-app20151023161601574
---

chrome 插件这里不细说，阅读此文需要有基本的插件基础

chrome插件调用本地程序和普通插件有些不一样，主要是分成三部分组成，除了需要插件本身外，还需要一个存放在本地的配置文件作为插件通讯基础，还有要被执行的本地程序

### 先介绍配置文件：
配置文件可以存放于`/etc/opt/chrome/native-messaging-hosts`或者`$HOME/.config/google-chrome/NativeMessagingHosts`
看需要选择其中一个目录存放即可（建议是放在后者）

配置文件基本结构如下：

```json
// 假如这个文件名字叫做 org.deepin.dmovie.json
{
    "name": "org.deepin.dmovie",
    "description": "play video with local app",
    "path": "/home/choldrim/.cache/play-video-locally/start",
    "type": "stdio",
    "allowed_origins": [
        "chrome-extension://geddpfooejohhebhamibjkgkcahnkbnd/"
    ]
}
```

**name:** 通讯时需要让插件知道往那个地方连接，这个字段就是连接时使用的唯一标志(注意：这个名字要和文件名字相同)

**description:** 描述字段
**path:** local app，本地可执行文件的路径
**type:** 使用何种方式通讯，stdio是指使用标准输入输出进行通讯（我写这篇博客的时候google只支持一种通讯方式）
**allowed_origins:** 允许访问通讯的插件连接，`geddpfooejohhebhamibjkgkcahnkbnd`是允许通讯的插件的ID

### 通讯所需配置文件准备好了，接下来准备本地可执行文件
我是用python3编写的一个本地程序，代码如下：
```python
#!/usr/bin/python3

import os
import sys
import struct
import json
import logging

# log what happen in background for debug your app
log = logging.getLogger("default")
handler = logging.FileHandler("/tmp/luanch_local_app.log")
formatter = logging.Formatter("%(asctime)s %(levelname)s %(message)s")
handler.setFormatter(formatter)
log.addHandler(handler)
log.setLevel(logging.DEBUG)

# get length of msg
length_bytes = sys.stdin.buffer.read(4)
if len(length_bytes) == 0:
    log.fatal("Err: fail to get msg length")
    quit(0)

# get msg
length = struct.unpack("i", length_bytes)[0]
text_json = sys.stdin.buffer.read(length).decode("utf-8")
log.debug("msg: %s" % text_json)
url = json.loads(text_json).get("text")

# start the target app
run_cmd = "echo 'hello chrome extension: %s'" % url
log.debug("run: %s" % run_cmd)

# luanch local app
os.system(run_cmd)
```

因为通讯使用stdio的方式，所以使用了 `sys.stdin.buffer.read()` 读取插件发送过来的内容
需要注意的是：**消息头部有4个字节是对消息长度的描述，需要先单独读取**

### 最后介绍插件
先贴一下主要的js代码
- popup.js

```javascript
var port = null;
var host_name = "org.deepin.dmovie";
var tab_url = "";

open_local_player = function(url){
    if (!port){
        connect();
    }
    port.postMessage({"text": url});
}

connect = function(){
    port = chrome.runtime.connectNative(host_name);
    port.onDisconnect.addListener(onDisconnected);
}

onDisconnected = function() {
    console.log("disconnect");
    port = null;
}

$("#defaultLinks").click(function(){
    chrome.tabs.query({active: true}, function(tab){
        url = tab[0].url;
        console.log(url);
        open_local_player(url)
        window.close();
    });
});
```

这是插件的popup的代码，defaultLinks是popup菜单的id，连接本地时，使用的是
`chrome.runtime.connectNative()`函数，以上代码要做的是启动本地app，并将tab的url发送到本地app

- mainifest.json

```json
{
    "manifest_version": 2,
    "name": "臣妾做不到...",
    "version": "2.0.0",
    "description": "capture movie from website",
    "permissions": [
        "nativeMessaging",
        "tabs"
    ],

    "icons":{
        "48": "images/deepin-movie_48.png"
    },

    "browser_action":{
        "default_icon":"images/deepin-movie_48.png",
        "default_popup":"pages/popup.html"
    }
}
```
注意：插件需要nativeMessaging权限

### 总结
(以上代码存放在我的github项目里 [play-video-locally](https://github.com/choldrim/play-video-locally), 欢迎 fork ;) )
个人感觉插件启动本地程序不算太难理解，本文主要是提供一个demo供大家参考，避免浪费太多时间在连接上，把更多的时间花在你的代码逻辑，去实现你更棒的想法吧 ;)
如果你也有好想法，并且不介意分享的话，请到下面评论也给我分享一下你的想法，谢谢（哈）

部分参考链接： 
[Native Messaging](https://developer.chrome.com/extensions/nativeMessaging)
[PointDownload](https://github.com/pointteam/pointdownload/blob/develop/PointChromeExtension%2FExtension%2Fpopup.js)
[Match-yang blog](http://match-yang.blog.163.com/blog/static/2109902542014319103739996)
