---
title: csrf(xsrf)原理与应对策略
date: 2015-12-19 16:21:13
tags:
    - web安全
    - Python
categories: 基础原理
---

>  **跨站请求伪造**（英语：Cross-site request forgery），也被称为 **one-click attack** 或者 **session riding**，通常缩写为 **CSRF** 或者 **XSRF**， 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。

<!--more-->
## 进攻方式

攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己曾经认证过(携带Session)的网站并执行一些操作（如发邮件，发消息，甚至财产操作如转账和购买商品）。由于浏览器曾经认证过，所以被访问的网站会认为是真正的用户操作而去执行。这利用了web中用户身份验证的一个漏洞：**简单的身份验证只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的**。
## 防御措施
1. 验证码
2. 检查Referer

   一般向`www.xxx.com`发送请求的referer也是`www.xxx.com`，而如果是CSRF攻击传来的请求，Referer字段会是包含恶意网址的地址，不会位于`www.xxx.com`之下，这时候服务器就能识别出恶意的访问。
3. Token

   由于CSRF的本质在于攻击者欺骗用户去访问自己设置的地址，所以如果要求在访问敏感数据请求时，要求用户浏览器提供不保存在cookie中，并且攻击者无法伪造的数据作为校验，那么攻击者就无法再执行CSRF攻击。这种数据通常是表单中的一个数据项。服务器将其生成并附加在表单中，其内容是一个伪乱数。当客户端通过表单提交请求时，这个伪乱数也一并提交上去以供校验。正常的访问时，客户端浏览器能够正确得到并传回这个伪乱数，而通过CSRF传来的欺骗性攻击中，攻击者无从事先得知这个伪乱数的值，服务器端就会因为校验token的值为空或者错误，拒绝这个可疑请求。
## Tornado中的做法

在Application的settings中添加

``` Python
"cookie_secret": "xxxxxxxxxxxxx"`
"xsrf_cookies": True
```

就会自动在非`get, head, option`请求中检查`_xsrf`参数是否存在且是否正确

而这个参数是通过以下函数产生的

``` python
    def _get_raw_xsrf_token(self):
        """Read or generate the xsrf token in its raw form.
        """
        if not hasattr(self, '_raw_xsrf_token'):
            cookie = self.get_cookie("_xsrf")
            if cookie:
                version, token, timestamp = self._decode_xsrf_token(cookie)
            else:
                version, token, timestamp = None, None, None
            if token is None:
                version = None
                token = os.urandom(16)
                timestamp = time.time()
            self._raw_xsrf_token = (version, token, timestamp)
        return self._raw_xsrf_token
```

然后在`finish`的时候回触发`check_xsrf_cookie`

``` python
    def check_xsrf_cookie(self):
        token = (self.get_argument("_xsrf", None) or
                 self.request.headers.get("X-Xsrftoken") or
                 self.request.headers.get("X-Csrftoken"))
        if not token:
            raise HTTPError(403, "'_xsrf' argument missing from POST")
        _, token, _ = self._decode_xsrf_token(token)
        _, expected_token, _ = self._get_raw_xsrf_token()
        if not _time_independent_equals(utf8(token), utf8(expected_token)):
            raise HTTPError(403, "XSRF cookie does not match POST argument")

```

这有什么用呢，之前我一直想写一个单元测试，这也是为了写单元测试，发送请求时候绕开xsrf时候所写的，通过伪造一个相同的函数就可以绕开了。
