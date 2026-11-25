---
title: Bugku charlottesweb WP
date: 2026-11-25 11:00:00
tags: [Bugku,Web,HTTP方法,PUT]
categories: [CTF-WP]
---

# Bugku charlottesweb
## 题目信息
- 题目名称：charlottesweb
- 考点：HTTP 请求方法（PUT）、405 Method Not Allowed
- 目标路由：`/super-secret-route-nobody-will-guess`

## 解题过程
1. 访问网址页面只有一个按钮，F12查看网页源码，发现注释里提示 `/src`，访问：
`http://49.232.142.230:18626/src`
查看源代码，找到路由 `/super-secret-route-nobody-will-guess` 存在flag。

2. 直接浏览器访问该路由，浏览器默认发起 `GET` 请求，服务器返回 `405 Method Not Allowed`，代表该路由不接受GET方法。

3. 源码提示：`PUT /super-secret-route-nobody-will-guess HTTP/1.1`，接口仅允许PUT请求。

4. 下面几种方式均可发包拿到flag：

### curl 发包
```bash
curl -X PUT http://49.232.142.230:18626/super-secret-route-nobody-will-guess
