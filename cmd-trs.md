title: 简单实现命令行使用有道翻译
date: 2016-02-21 09:05:48
tags: [python,translation]
category: linux-cmdline
thumbnail: http://7xrkyd.com1.z0.glb.clouddn.com/distcc%2Fblog_ppb_s.jpg
---

## 先贴几张效果图吧
![中译英](http://7xrkyd.com1.z0.glb.clouddn.com/cmd-trs%2F1.png)
![中译英(句子)](http://7xrkyd.com1.z0.glb.clouddn.com/cmd-trs%2F2.png)
![英译中](http://7xrkyd.com1.z0.glb.clouddn.com/cmd-trs%2F3.png)
![英译中(句子)](http://7xrkyd.com1.z0.glb.clouddn.com/cmd-trs%2F4.png)

## 材料：
## 有道 api-key
http://fanyi.youdao.com/openapi?path=data-mode
网站名称、网站地址、说明随便填写东西就可以了，有道并无人工验证
如：
![](http://7xrkyd.com1.z0.glb.clouddn.com/cmd-trs%2F5.png)

## 依赖
```shell
sudo pip3 install requests
```

## 完整源码
```python
import os
import sys

from configparser import ConfigParser

import requests

HOST = "http://fanyi.youdao.com/openapi.do"
CONF_FILE = "%s/.config/youdao-trs/conf" % os.getenv("HOME")

def get_conf():
    config = ConfigParser()
    config.read(CONF_FILE)
    api_key = config["default"]["APIKey"]
    key_from = config["default"]["Keyfrom"]
    return api_key, key_from

def trs(word):
    api_key, key_from = get_conf()
    url = "%s?keyfrom=%s&key=%s&type=data&doctype=json&version=1.1&q=%s" %(HOST, key_from, api_key, word)
    r = requests.get(url)
    data = r.json()
    print("翻译：")
    print("    %s" % "\n".join(data.get("translation")))
    basic = data.get("basic")
    if basic:
        if "phonetic" in basic:
            print("发音：")
            print("    %s" % basic.get("phonetic"))
            print()
            if "us-phonetic" in basic:
                print("发音(us)：")
                print("    %s" % basic.get("us-phonetic"))
            if "uk-phonetic" in basic:
                print("发音(uk)：")
                print("    %s" % basic.get("uk-phonetic"))


if __name__ == "__main__":
    words = " ".join(sys.argv[1:])
    trs(words)
```

配置文件，模板
```ini
[default]
APIkey = $you-key
Keyfrom= $keyfrom
```

