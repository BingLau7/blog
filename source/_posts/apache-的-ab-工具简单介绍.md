---
title: apache 的 ab 工具简单介绍
date: 2015-11-13 16:20:33
tags:
    - http
categories: 工具
---

首先我们看看 `ab -h` 命令提供的信息:

``` shell
Usage: ab [options] [http[s]://]hostname[:port]/path
Options are:
    -n requests     Number of requests to perform # 请求数
    -c concurrency  Number of multiple requests to make at a time # 并发请求数
    -t timelimit    Seconds to max. to spend on benchmarking # 测试所进行的最大秒数。
                    This implies -n 50000 # 其内部隐含值是-n 50000
    -s timeout      Seconds to max. wait for each response # 响应等待超时时间
                    Default is 30 seconds # 默认是 30 秒
    -b windowsize   Size of TCP send/receive buffer, in bytes # TCP窗口大小
    -B address      Address to bind to when making outgoing connections # 代理地址？
    -p postfile     File containing data to POST. Remember also to set -T # post请求包含的文件，也可以使用 -T
    -u putfile      File containing data to PUT. Remember also to set -T # put求包含的文件，也可以使用 -T
    -T content-type Content-type header to use for POST/PUT data, eg. # post / put 请求时候 content-type 请求头
                    'application/x-www-form-urlencoded'
                    Default is 'text/plain'
    -v verbosity    How much troubleshooting info to print # 是否需要打印故障日志
    -w              Print out results in HTML tables # 以HTML表的格式输出结果
    -i              Use HEAD instead of GET # 使用 HEAD 方法代替 GET 方法
    -x attributes   String to insert as table attributes # 设置<table>属性的字符串。
    -y attributes   String to insert as tr attributes # 设置<tr>属性的字符串。
    -z attributes   String to insert as td or th attributes # 设置<td>属性的字符串。
    -C attribute    Add cookie, eg. 'Apache=1234'. (repeatable) # 对请求附加一个Cookie:行。其典型形式是name=value的一个参数对，此参数可以重复。
    -H attribute    Add Arbitrary header line, eg. 'Accept-Encoding: gzip' #对请求附加额外的头信息。此参数的典型形式是一个有效的头信息行，其中包含了以冒号分隔的字段和值的对(如,"Accept-Encoding:zip/zop;8bit")。
                    Inserted after all normal header lines. (repeatable)
    -A attribute    Add Basic WWW Authentication, the attributes # 对服务器提供BASIC认证信任。
                    are a colon separated username and password.
    -P attribute    Add Basic Proxy Authentication, the attributes # 对一个中转代理提供BASIC认证信任。
                    are a colon separated username and password.
    -X proxy:port   Proxyserver and port number to use # 对请求使用代理服务器。
    -V              Print version number and exit # 显示版本号
    -k              Use HTTP KeepAlive feature # 启用HTTP KeepAlive功能，即在一个HTTP会话中执行多个请求。默认时，不启用KeepAlive功能。
    -d              Do not show percentiles served table. # 不显示"percentage served within XX [ms] table"的消息(为以前的版本提供支持)。
    -S              Do not show confidence estimators and warnings. # 不显示估计配置与警告
    -q              Do not show progress when doing more than 150 requests # 如果处理的请求数大于150，ab每处理大约10%或者100个请求时，会在stderr输出一个进度计数。此-q标记可以抑制这些信息。
    -l              Accept variable document length (use this for dynamic pages) # 接受可变长度的文档
    -g filename     Output collected data to gnuplot format file. # 通过 gnuplot 文件格式收集输出数据
    -e filename     Output CSV file with percentages served # 产生一个以逗号分隔的(CSV)文件，其中包含了处理每个相应百分比的请求所需要(从1%到100%)的相应百分比的(以微妙为单位)时间。由于这种格式已经“二进制化”，所以比'gnuplot'格式更有用。
    -r              Don't exit on socket receive errors. # 当 socket 返回错误时不退出
    -m method       Method name # 请求方法名称
    -h              Display usage information (this message) # 显示帮助信息
    -Z ciphersuite  Specify SSL/TLS cipher suite (See openssl ciphers) # 指定 SSL/TLS 加密方式
    -f protocol     Specify SSL/TLS protocol # 指定 SSL/TLS 协议
                    (SSL3, TLS1, TLS1.1, TLS1.2 or ALL)
```

接下来我们使用 `http://www.douban.com/` 作为测试网站

``` shell
➜  ~ ab -n 100 -c 100 http://www.douban.com/
This is ApacheBench, Version 2.3 <$Revision: 1604373 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking www.douban.com (be patient).....done


Server Software:        dae                         # 测试服务器软件名称
Server Hostname:        www.douban.com               # 测试网址Host
Server Port:            80                          # 测试端口

Document Path:          /                           # 测试Path
Document Length:        160 bytes                    # 返回的报文大小

Concurrency Level:      100                         # 标识并发的用户数
Time taken for tests:   0.284 seconds                # 执行完所有的请求所花费的时间
Complete requests:      100                         # 完成请求数
Failed requests:        0                           # 失败请求数
Non-2xx responses:      100                         # 返回响应非2xx数
Total transferred:      30100 bytes                  # 整个过程中的网络传输量
HTML transferred:       16000 bytes                  # 整个过程中的HTML大小
Requests per second:    352.02 [#/sec] (mean)         # 吞吐率,表示每秒处理的请求数，mean标识平均值
Time per request:       284.075 [ms] (mean)           # 每个用户等待时间，mean标识平均值
Time per request:       2.841 [ms] (mean, across all concurrent requests) #服务器平均请求处理的时间. 
Transfer rate:          103.47 [Kbytes/sec] received   # 这些请求在单位内,从服务器获取的数据长度.

Connection Times (ms)                               # 网络上消耗的时间的分解
              min  mean[+/-sd] median   max
Connect:       26   37   7.9     35      63
Processing:    44  130  76.7     88     239
Waiting:       44  129  76.5     88     239
Total:         70  167  77.5    133     277

Percentage of the requests served within a certain time (ms) # 整个场景中所有请求的响应情况。
  50%    133
  66%    240
  75%    271
  80%    272
  90%    275
  95%    277
  98%    277
  99%    277
 100%    277 (longest request)
```
