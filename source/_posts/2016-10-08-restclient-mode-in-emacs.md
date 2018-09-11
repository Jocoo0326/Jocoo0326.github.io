---
layout: post
title: Emacs中的restclient-mode
date: 2016-10-08
categories:
- tools
tags:
- restclient
---

开发中常常需要debug后台的service工具，比如ruby实现的rest-client，intellij-idea自带rest client tool，vim编辑器也有vim-http-client，今天要介绍的是emacs中的restclient-mode
<!-- more -->
emacs中的restclient-mode支持请求头参数配置，添加请求体数据，格式化xml/json数据，同时还支持显示图片，前提是编译时加入图片库的功能，Linux中默认配置，但是Windows对应版本却没有，官方下载的emacs也不行，所以需要自己编译Emacs，参见[Windows中编译emacs 64bit](/2016/09/06/2016-9-6-building-emacs-w64-with-msys2/)

首先从melpa中安装restclient，至于国内使用的elpa源参见我的[Emacs配置文件](https://github.com/Jocoo0326/emacs.d)
然后M-x启动restclient-mode

常用快捷键：
* C-c C-c: runs the query under the cursor, tries to pretty-print the response (if possible)
* C-c C-r: same, but doesn't do anything with the response, just shows the buffer
* C-c C-v: same as C-c C-c, but doesn't switch focus to other window

### example
Get请求并设置请求头
```
GET https://api.github.com
User-Agent: Emacs Restclient
```
Response:
``` json
{
  "current_user_url": "https://api.github.com/user",
  "current_user_authorizations_html_url": "https://github.com/settings/connections/applications{/client_id}",
  "authorizations_url": "https://api.github.com/authorizations",
  "code_search_url": "https://api.github.com/search/code?q={query}{&page,per_page,sort,order}",
  "commit_search_url": "https://api.github.com/search/commits?q={query}{&page,per_page,sort,order}",
  "emails_url": "https://api.github.com/user/emails",
  "emojis_url": "https://api.github.com/emojis",
  "events_url": "https://api.github.com/events",
  "feeds_url": "https://api.github.com/feeds",
  "followers_url": "https://api.github.com/user/followers",
  "following_url": "https://api.github.com/user/following{/target}",
  "gists_url": "https://api.github.com/gists{/gist_id}",
  "hub_url": "https://api.github.com/hub",
  "issue_search_url": "https://api.github.com/search/issues?q={query}{&page,per_page,sort,order}",
  "issues_url": "https://api.github.com/issues",
  "keys_url": "https://api.github.com/user/keys",
  "notifications_url": "https://api.github.com/notifications",
  "organization_repositories_url": "https://api.github.com/orgs/{org}/repos{?type,page,per_page,sort}",
  "organization_url": "https://api.github.com/orgs/{org}",
  "public_gists_url": "https://api.github.com/gists/public",
  "rate_limit_url": "https://api.github.com/rate_limit",
  "repository_url": "https://api.github.com/repos/{owner}/{repo}",
  "repository_search_url": "https://api.github.com/search/repositories?q={query}{&page,per_page,sort,order}",
  "current_user_repositories_url": "https://api.github.com/user/repos{?type,page,per_page,sort}",
  "starred_url": "https://api.github.com/user/starred{/owner}{/repo}",
  "starred_gists_url": "https://api.github.com/gists/starred",
  "team_url": "https://api.github.com/teams",
  "user_url": "https://api.github.com/users/{user}",
  "user_organizations_url": "https://api.github.com/user/orgs",
  "user_repositories_url": "https://api.github.com/users/{user}/repos{?type,page,per_page,sort}",
  "user_search_url": "https://api.github.com/search/users?q={query}{&page,per_page,sort,order}"
}
// GET https://api.github.com
// HTTP/1.1 200 OK
// Date: Mon, 27 Aug 2018 07:37:54 GMT
// Content-Type: application/json; charset=utf-8
// Content-Length: 2039
// Server: GitHub.com
// Status: 200 OK
// X-RateLimit-Limit: 60
// X-RateLimit-Remaining: 58
// X-RateLimit-Reset: 1535359056
// Cache-Control: public, max-age=60, s-maxage=60
// Vary: Accept
// ETag: "7dc470913f1fe9bb6c7355b50a0737bc"
// X-GitHub-Media-Type: github.v3; format=json
// Access-Control-Expose-Headers: ETag, Link, Retry-After, X-GitHub-OTP, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, X-OAuth-Scopes, X-Accepted-OAuth-Scopes, X-Poll-Interval
// Access-Control-Allow-Origin: *
// Strict-Transport-Security: max-age=31536000; includeSubdomains; preload
// X-Frame-Options: deny
// X-Content-Type-Options: nosniff
// X-XSS-Protection: 1; mode=block
// Referrer-Policy: origin-when-cross-origin, strict-origin-when-cross-origin
// Content-Security-Policy: default-src 'none'
// X-Runtime-rack: 0.015700
// Vary: Accept-Encoding
// X-GitHub-Request-Id: 058C:2AEB:8AD659:BCC8BE:5B83AA40
// Request duration: 0.545086s
```

Post请求, 设置请求头并提交json数据，json数据之前要有空行
```
POST https://jira.atlassian.com/rest/api/2/search
Content-Type: application/json

{
    "jql": "project = HSP",
    "startAt": 0,
    "maxResults": 15,
    "fields": [
        "summary",
        "status",
        "assignee"
    ]
}
```

GET请求还能够显示图片
```
# show image
GET http://f.hiphotos.baidu.com/image/pic/item/e824b899a9014c08bbd92197077b02087bf4f426.jpg
```
Tips： restclient-mode以注释为请求的分割线(#开头)，再添加请求的时候最好加上注释

### 变量
请求中常常有某些参数是相同的，此时可以定义一个buffer内变量
```
:user_name = Jocoo
```
还可以多行变量
```
:var = <<
Authorization: :my-auth
name: :user_name
Content-Type: application/json
User-Agent: SomeApp/1.0
#
```
多行变量还可以执行elisp代码
```
:myvar := <<
(some-long-elisp
    (code spanning many lines)
#
```
通常的用法
```
# Some generic vars

:my-auth = 319854857345898457457
:my-headers = <<
Authorization: :my-auth
Content-Type: application/json
User-Agent: SomeApp/1.0
#

# Update a user's name

:user-id = 7
:the-name := (format "%s %s %d" 'Neo (md5 "The Chosen") (+ 100 1))

PUT http://localhost:4000/users/:user-id/
:my-headers

{ "name": ":the-name" }
```