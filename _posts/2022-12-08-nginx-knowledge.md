---
title: Nginx 知识汇总
author: zhangzangqian
date: 2022-12-08 14:00:00 +0800
categories: [技术]
tags: [Nginx]
math: true
mermaid: true
---

## location 语法

```conf
location [=|~|~*|^~] /uri/ { ... }
```

1.  =
    
    表示精确匹配，优先级最高，匹配成功后则停止向下搜索。


    ```conf
    # 精确匹配，必须是 127.0.0.1/
    location = / {
        ...
    }
    # 精确匹配，必须是 127.0.0.1/login
    location = /login {
        ...
    }
    ```

2. ^~

    对 uri ***起始字符*** 做 ***字符串匹配***，不是 ***正则匹配***。区分大小写。

    ```conf
    # 127.0.0.1/static/js
    location ^~ /static/ {
        ...
    }
    ``` 

3. ~

    对 uri （可以不是起始字符串）做 ***正则匹配***，区分大小写。

    ```conf
    # 区分大小写，以 gif,jpg,js结尾
    location ~ \.(gif|jpg|png|js|css)$ {
        ...
    }
    ```

4. ~*

    对 uri （可以不是起始字符）做 ***正则匹配***，不区分大小写。

    ```conf
    # 不区分大小写，匹配.png结尾的。
    location ~* \.png$ {
        ...
    }
    ```

5. !~ 和 !~*

    都是***非xx***得***正则匹配***，但是 !~ 区分大小写，!~* 不区分大小写。

    ```conf
    # 区分大小写，匹配不以.xhtml结尾的
    location !~ \.xhtml$ {
        ...
    }

    # 不区分大小写，匹配不以.xhtml结尾的
    location !~* \.xhtml$ {
        ...
    }
    ```

## location 和 proxy_pass 是否以 / 结尾

1. 在 location 中匹配的 url 最后有无 / 结尾，指的是模糊匹配与精确匹配的问题。

    1.1 没有 / 结尾时，location /abc/def 可以匹配 /abc/defghi 的请求，也可以匹配 /abc/def/ghi ......

    1.2 有 / 结尾时,location /abc/def/ 不能匹配/abc/defghi的请求，只能精确匹配 /abc/def/ghi这样的请求

2. 在 proxy_pass 中代理的 url 最后有无 / 结尾，指的是在 proxy_pass 指定的 url 后要不要加上 location 匹配的 url 的问题。

    举例： http://xpzzd.com/proxy/login.html

    ```conf
    # 情况1
    # proxy_pass的 最终地址就是: http://xpzzd.com:8000/login.html  因为 proxy_pass 以 / 跟结尾，代表绝对路径，所以不会加上 location 匹配的 proxy
    location /proxy/ {
        proxy_pass http://xpzzd.com:8000/;
    }

    # 情况2
    # proxy_pass 代理到 http://xpzzd.com:8000/proxy/login.html
    location /proxy/ {
        proxy_pass http://xpzzd.com:8000;
    }

    # 情况3
    # proxy_pass 代理到 http://xpzzd.com:8000/disquz/login.html
    location /proxy/ {
        proxy_pass http://xpzzd.com:8000/disquz/;
    }

    # 情况4
    # proxy_pass 代理到 http://xpzzd.com:8000/disquzlogin.html
    location /proxy/ {
        proxy_pass http://xpzzd.com:8000/disquz;
    }
    ```

## 修改上传文件大小
    
1. 修改 `nginx.conf` 文件中 `http` 节点下 `client_max_body_size 1024m`;
2. 执行命令 `nginx -s reload`
