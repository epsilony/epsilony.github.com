---
layout: post
title: "在Python中使用有道翻译 Using Youdao translation API through Python3"
description: ""
category: miscellaneous note
tags: [Python, Youdao Fanyi api, 有道翻译api]
---
{% include JB/setup %}

有道翻译大概是目前最佳的云词典/翻译方案了，免费提供最多1k/h字的查询。

先去[有道翻译API](http://fanyi.youdao.com/openapi)弄个调用数据接口的 APP KEY

_MS 不用等审，随填随给_ 还算不错

演示代码：
<!--more-->

{% highlight python %}
API_KEY="YOUR_API_KEY"
SITE_NAME="YOUR_SITE_NAME"

import http.client as clt
import urllib
import json

QUERY="/openapi.do?keyfrom=%s&key=%s&type=data&doctype=json&version=1.1&q=%s"

YOUDAO_FANYI="fanyi.youdao.com"

def query_url(query,**kwargs):
    url = QUERY%(SITE_NAME,API_KEY,urllib.request.quote(query))
    only=kwargs.get("only")
    if only:
        url+="&only="+only
    return url

def query_youdao(query,**kwargs):
    conn=clt.HTTPConnection(YOUDAO_FANYI)
    conn.request("GET",query_url(query,**kwargs))
    response=conn.getresponse()
    result=json.loads(response.read().decode('utf8'))
    conn.close()
    return result

print(query_youdao("hello world"))
{% endhighlight %}

在设置完`API_KEY`和`SITE_NAME`后运行的结果：
{% highlight python %}
{'basic': {'explains': ['你好世界']}, 'query': 'hello world', 'web': [{'value': ['你好世界', '举个例子', '开始'], 'key': 'hello world'}, {'value': ['会写的人多了去了'], 'key': 'Hello   World'}, {'value': ['凯蒂猫气球世界'], 'key': 'Hello Kitty World'}], 'translation': ['你好,世界'], 'errorCode': 0}
{% endhighlight %}
