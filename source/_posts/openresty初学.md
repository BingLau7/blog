---
title: openresty初学
date: 2016-10-01 16:27:20
tags:
    - openresty
    - nginx
    - lua
categories: 工具
---

## 背景

前段时间由于不想将一些团队必须有的业务逻辑（如封禁IP，各种常规检查，转发等）写于各个业务线上，于是我们就动起了使用`lua`在 `nginx`做手脚的心思，一开始本来就组长一个人在弄，后面我出于好奇，就『自愿』援助了其中一条业务线的一个适合写在`nginx`的功能，这个功能具体不加以描述，总体来说涉及到了过滤请求，查询数据库，分发请求这几个步骤。

<!-- more -->

另说明，这里只是介绍一下简单的`OpenRestry`应用，本片文章并不涉及其太过深入的阐述，其中介绍的方法，库也都是常用的几个方法，而非所有都会有所涉及，希望这篇文章给你带来一些启迪，如果你想了解更多更加希望你能去看官方文档并参与其社区讨论。
## `OpenResty`简介

[OpenResty/lua-nginx-module](https://github.com/openresty/lua-nginx-module#readme)

>  OpenResty 是中国人章亦春发起的一个开源项目，它的核心是基于 NGINX 的一个 C 模块，该模块将 Lua 语言嵌入到 NGINX 服务器中，并对外提供一套完整 Lua Web 应用开发 API，透明地支持非阻塞 I/O，提供了“轻量级线程”、定时器等等高级抽象，同时围绕这个模块构建了一套完备的测试框架、调试技术以及由 Lua 实现的周边功能库；这个项目的**意义在于极大的降低了高性能服务端的开发难度和开发周期**，在快节奏的互联网时代这一点极为重要。

这里是[infoQ一篇文章](http://www.infoq.com/cn/articles/what-is-openresty-mentioned-in-smartisan-release-conference)的节选介绍。简单来说`OpenResty`给我们在`nginx`中提供了一个`lua`的钩子，方便我们通过`lua`在`nignx`这一个web入口层实现一些相对`nginx`配置的复杂的功能（但由于`lua`语言的局限性，我个人不推荐在这里实现太过复杂的业务逻辑）。

`ngx_openresty` 目前有两大应用目标：
1. 通用目的的 web 应用服务器。在这个目标下，现有的 web 应用技术都可以算是和 OpenResty 或多或少有些类似，比如 Nodejs, PHP 等等。ngx_openresty 的性能（包括内存使用和 CPU 效率）算是最大的卖点之一。
2. Nginx 的脚本扩展编程，用于构建灵活的 Web 应用网关和 Web 应用防火墙。有些类似的是 NetScaler。其优势在于 Lua 编程带来的巨大灵活性。
## Lua入门

[Lua简明教程](http://coolshell.cn/articles/10739.html)
## Lua处理在Nginx的几个阶段

### init_by_lua

应用模块：**http**

主要用于加载时候的初始化，在nginx重新加载配置文件时候，就会运行。

### set_by_lua

应用模块：**server, location**

主要用于设置变量，常用于计算一个逻辑，然后返回结果，此时不能使用`Output API、Control API、Subrequest API、Cosocket API`

### rewrite_by_lua

应用模块：**http, server, location**

此处应该与nginx中的`NGX_HTTP_REWRITE_PHASE`阶段对应，是在寻找到匹配的`location`之后用于修改请求的`URI`，在访问前执行。

### access_by_lua

应用模块：**http, server, location**

主要用于访问控制，能收集到大部分变量，类似status需要在log阶段才有。

这条指令运行于nginx access阶段的末尾，因此总是在 allow 和 deny 这样的指令之后运行，虽然它们同属 access 阶段。

### content_by_lua

应用模块：**location**

阶段是所有请求处理阶段中最为重要的一个，运行在这个阶段的配置指令一般都肩负着生成内容（content）并输出HTTP响应。

### header_filter_by_lua

应用模块：**http, server, location**

一般只用于设置Cookie和Headers等。

### body_filter_by_lua

应用模块：**http, server, location**

一般会在一次请求中被调用多次, 因为这是实现基于 HTTP 1.1 chunked 编码的所谓“流式输出”的。

### log_by_lua

应用模块：**http, server, location**

该阶段总是运行在请求结束的时候，用于请求的后续操作，如在共享内存中进行统计数据,如果要高精确的数据统计，应该使用body_filter_by_lua。

## Hello World

这里忽略下载步骤，仅仅流程化地提一下 Hello World。

先写下如下的ngixn配置

``` nginx
worker_processes  1;
error_log logs/error.log;
events {
    worker_connections 1024;
}
http {
    server {
        listen 8080;
        location / {
            default_type text/html;
            content_by_lua '
                ngx.say("<p>hello, world</p>")
            ';
        }
    }
}
```

然后在启动nginx时候加入`-c`参数指明该配置文件所在位置应该就能顺利启动了（记住是openresty的nginx启动命令）。

## 获取当前请求

请求信息是被openresty封装在了`ngx.req.*`中，文档中有着比较详尽的[描述](https://github.com/openresty/lua-nginx-module#ngxreqis_internal)

``` lua
function get_request_info()
  local headers = ngx.req.get_headers()  -- 获取头描述，返回一个table
  local uri = ngx.var.uri                -- 获取其请求的uri
  local method = ngx.req.get_method()    -- 获取请求方法字符串
  local args = ngx.req.get_uri_args()    -- 获取uri中的参数，返回一个table
  ngx.req.read_body()                    -- 指明需要读取body，脱离Nginx事件模块阻塞去同步读取请求body
  local body = ngx.req.get_body_data()   -- 获取请求体                    

  local request_info = {
    headers=headers,
    uri=uri,
    args=args,
    method=method,
    body=body,
  }

  return request_info
end

local cjson = require "cjson"
ngx.log(ngx.WARN, cjson.encode(get_request_info()))
```

`nginx.conf`:

``` nginx
worker_processes  1;
error_log logs/error.log;
events {
    worker_connections 1024;
}
http {
    server {
        listen    8080;

        location /test {
            content_by_lua_file /Users/binglau/code/learn/openresty_work/conf/test.lua;
        }
     }

}
```

请求：

``` shell
> curl localhost:8080/test?test=123 -v
*   Trying ::1...
* connect to ::1 port 8080 failed: Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /test?test=123 HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.43.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: openresty/1.9.15.1
< Date: Tue, 27 Sep 2016 12:05:52 GMT
< Content-Type: text/plain
< Transfer-Encoding: chunked
< Connection: keep-alive
<
* Connection #0 to host localhost left intact
```

日志：

``` shell
2016/09/27 20:05:52 [error] 51223#0: *5 [lua] test.lua:23: {"method":"GET","uri":"\/test","args":{"test":"123"},"headers":{"host":"localhost:8080","accept":"*\/*","user-agent":"curl\/7.43.0"}}, client: 127.0.0.1, server: , request: "GET /test?test=123 HTTP/1.1", host: "localhost:8080"
```
## 数据库操作

这里简单说一下`lua`的数据库操作，与前面的配置类似，就只是介绍:
1. 创建一个新Mysql实例：

   ``` lua
   local mysql = require "resty.mysql"
   local db, err = mysql:new()
   if not db then
    ngx.say("failed to instantiate mysql: ", err)
    return
   end
   ```
2. 发起连接：

   ``` lua
   local ok, err, errcode, sqlstate = db:connect{
    host = "127.0.0.1",
    port = 3306,
    database = "ngx_test",
    user = "ngx_test",
    password = "ngx_test",
    max_packet_size = 1024 * 1024 }

   if not ok then
    ngx.say("failed to connect: ", err, ": ", errcode, " ", sqlstate)
    return
   end
   ```
3. 执行sql语句

   ``` lua
   -- syntax: res, err, errcode, sqlstate = db:query(query)
   -- syntax: res, err, errcode, sqlstate = db:query(query, nrows)
   res, err, errcode, sqlstate =
    db:query("insert into cats (name) "
               .. "values (\'Bob\'),(\'\'),(null)")
   if not res then
    ngx.say("bad result: ", err, ": ", errcode, ": ", sqlstate, ".")
    return
   end

   ngx.say(res.affected_rows, " rows inserted into table cats ",
          "(last insert id: ", res.insert_id, ")")

   -- run a select query, expected about 10 rows in
   -- the result set:
   res, err, errcode, sqlstate =
    db:query("select * from cats order by id asc", 10)
   if not res then
    ngx.say("bad result: ", err, ": ", errcode, ": ", sqlstate, ".")
    return
   end
   ```
4. 长连接操作

   ``` lua
   -- put it into the connection pool of size 100,
   -- with 10 seconds max idle timeout
   local ok, err = db:set_keepalive(10000, 100)
   if not ok then
    ngx.say("failed to set keepalive: ", err)
    return
   end
   ```
5. 简单防止注入

   ``` lua
   local name = ngx.unescape_uri(ngx.var.arg_name)
   local quoted_name = ngx.quote_sql_str(name)
   local sql = "select * from users where name = " .. quoted_name
   ```
## 发送请求

这里先介绍一下一个`openresty`的内置方法`capture`

``` lua
-- syntax: res = ngx.location.capture(uri, options?)
-- context: rewrite_by_lua*, access_by_lua*, content_by_lua*

location /test {
       content_by_lua_block {
           local res = ngx.location.capture(
                    '/print_param',
                    {
                       method = ngx.HTTP_POST,
                       args = ngx.encode_args({a = 1, b = '2&'}),
                       body = ngx.encode_args({c = 3, d = '4&'})
                   }
                )
           ngx.say(res.body)
       }
   }
```

这里的`ngx.location.capture`其实是[发起一个子请求](https://github.com/openresty/lua-nginx-module#ngxlocationcapture)，而不是一个真正的 http 请求，在搜索如何发起一个新请求的时候大多数人都会告诉你一个库[lua-resty-http](https://github.com/pintsized/lua-resty-http)。

``` lua
http {
    server {
        listen    80;

        location /test {
            content_by_lua_block {
                ngx.req.read_body()
                local args, err = ngx.req.get_uri_args()

                local http = require "resty.http"   -- ①
                local httpc = http.new()
                local res, err = httpc:request_uri( -- ②
                    "http://127.0.0.1:81/spe_md5",
                        {
                        method = "POST",
                        body = args.data,
                      }
                )

                if 200 ~= res.status then
                    ngx.exit(res.status)
                end

                if args.key == res.body then
                    ngx.say("valid request")
                else
                    ngx.say("invalid request")
                end
            }
        }
    }

    server {
        listen    81;

        location /spe_md5 {
            content_by_lua_block {
                ngx.req.read_body()
                local data = ngx.req.get_body_data()
                ngx.print(ngx.md5(data .. "*&^%$#$^&kjtrKUYG"))
            }
        }
    }
}
```

这样你是可以发起一个新请求，起初我也是打算用这个库，最终看其文档始终不能完成其中的连接池功能，于是寻求到了另一个方法。

利用`proxy_pass`发起，请求到另外一个上游。

``` lua
http {
    upstream md5_server{
        server 127.0.0.1:81;
        keepalive 20;
    }

    server {
        listen    80;

        location /test {
            content_by_lua_block {
                ngx.req.read_body()
                local args, err = ngx.req.get_uri_args()

                local res = ngx.location.capture('/spe_md5',
                    {
                        method = ngx.HTTP_POST,
                        body = args.data
                    }
                )

                if 200 ~= res.status then
                    ngx.exit(res.status)
                end

                if args.key == res.body then
                    ngx.say("valid request")
                else
                    ngx.say("invalid request")
                end
            }
        }

        location /spe_md5 {
            proxy_pass http://md5_server; 
        }
    }

    server {
        listen    81; 

        location /spe_md5 {
            content_by_lua_block {
                ngx.req.read_body()
                local data = ngx.req.get_body_data()
                ngx.print(ngx.md5(data .. "*&^%$#$^&kjtrKUYG"))
            }
        }
    }
}
```

但是这样一来我们每一个子请求几乎都需要配置一个`proxy_pass`，不过我在网上又找到了[一个方法](http://www.cnblogs.com/kyrios/p/ngx_lua-httpclient-using-capture-and-proxy.html)

``` nginx
location /proxy/ {
     internal;
    rewrite ^/proxy/(https?)/([^/]+)/(.*)     /$3 break;
    proxy_pass      $1://$2;
}
```

通过`rewrite`的分组重写，我们动态配置`proxy_pass`，这样如果是请求`http://www.example.com/foo/bar`就只需要在lua代码中构造一个请求`/proxy/http/www.example.com/foo/bar`即可。

## 参考资料
1. [OpenResty文档](https://github.com/openresty/lua-nginx-module)
2. [OpenResty最佳实践](https://www.gitbook.com/book/moonbingbing/openresty-best-practices)
3. [OpenResty官网](https://openresty.org/cn/)
4. [ngx_lua学习笔记 -- capture + proxy](http://www.cnblogs.com/kyrios/p/ngx_lua-httpclient-using-capture-and-proxy.html)
