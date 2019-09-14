---
layout:     post
title:      "使用Golang模拟登录Github"
subtitle:   ""
date:       2019-09-14
author:     "SmartYi"
header-img: "img/in-post/login-github/head.jpeg"
tags:
    - Github
    - OAuth
    - Golang
---

## 使用Golang模拟登录Github
> - Oauth的流程请求登录->用户登录->用户授权->回调code->第三方登录成功
> - Github在登录页面html里隐藏authenticity_token，发送登录请求的时候需要带上
> - Golang中使用cookiejar模拟浏览器中的操作，记录连续请求的cookie

#### OAuth流程

![](https://pic-1255962289.cos.ap-guangzhou.myqcloud.com/20190914200951.png)

#### Golang实现模拟登录
0. HttpClient使用CookieJar记录请求过程中的Cookie值

```
jar, err := cookiejar.New(nil)
if err != nil {
    return &client
}
httpClient.Jar = jar
```

1. 请求LeetCode的登录页面`https://leetcode.com/accounts/github/login/?next=%2F`，会自动Redirect到github的登录页面，带着LeetCode的ClientId/RedirectURL等信息

```
req, err := http.NewRequest(http.MethodGet, LeetCodeLoginUrl, nil)
resp, err := client.DoRequest(req)
```

2. 在页面中获取`authenticity_token`，带上Github的用户名密码请求Github的登录`https://github.com/session`

```
req, err = http.NewRequest(http.MethodPost, GithubLoginSession,
    bytes.NewBuffer([]byte(MapToUrlParam(map[string]string{
        "commit":             "Sign in",
        "utf8":               "✓",
        "authenticity_token": token,
        "login":              GithubUsername,
        "password":           GithubPassword,
    }))))

headers := map[string]string{
    "Content-Type": `application/x-www-form-urlencoded`,
    "User-Agent":   `Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36`,
}
for k, v := range headers {
    req.Header.Set(k, v)
}
resp, err = client.DoRequest(req)
```

3. 如果是第一次登录，在Github上还没完成对LeetCode应用的授权，需要再次请求`https://github.com/login/oauth/authorize`完成授权

```
// token/clientId/redirectUri/redirectUri/state/scope等值均可以解析返回的HTML页面中获取
req, err = http.NewRequest(http.MethodPost, GithubOauthSession,
    bytes.NewBuffer([]byte(MapToUrlParam(map[string]string{
        "authorize":          "1",
        "authenticity_token": token,
        "utf8":               "✓",
        "client_id":          clientId,
        "redirect_uri":       redirectUri,
        "state":              state,
        "scope":              scope,
    }))))
headers := map[string]string{
    "Content-Type": `application/x-www-form-urlencoded`,
    "User-Agent":   `Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36`,
}
for k, v := range headers {
    req.Header.Set(k, v)
}

resp, err = client.DoRequest(req)
```

4. 如果已经完成授权，需要在页面中找到redirect url，带上Authorization code重定向回到LeetCode

```
req, err = http.NewRequest(http.MethodGet, redirectUrl, nil)
resp, err = client.DoRequest(req)
```

5. 最后LeetCode使用Authorization code换取Authorization Token，获取到我们在Github的账户信息

#### 参考博客

- [理解OAuth 2.0 - 阮一峰的网络日志](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
