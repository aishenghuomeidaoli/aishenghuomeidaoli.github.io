---
title: 搭建自己的GITMENT代理服务器
date: 2019-04-19 14:09:52
categories:
- 杂记
tags:
- Gitment
description: 搭建自己的GITMENT代理服务器,修复[object ProgressEvent]报错
keywords: 
- Gitment
---
## 背景
我用[hexo](https://hexo.io/)创建了个[博客](https://aishenghuomeidaoli.github.io/)，使用的主题是[NEXT](https://github.com/iissnan/hexo-theme-next)，并且在主题内配置了gitment评论功能。
然而最近评论功能失效，登录后提示\[object ProgressEvent\]就探索了下解决办法。

## 原因

一句话概括：实现gitment功能的代理服务，`SSL`证书失效了。下面是过程分析。

<!--more-->

在`next`主题的`gitment`[代码](https://github.com/iissnan/hexo-theme-next/blob/master/layout/_third-party/comments/gitment.swig#L4)里，可以看到根据主题内`gitment`的mint配置，主题会引用不同的js文件，代码如下：

```js
    {% if theme.gitment.mint %}
        {% set CommentsClass = "Gitmint" %}
        <link rel="stylesheet" href="https://aimingoo.github.io/gitmint/style/default.css">
        <script src="https://aimingoo.github.io/gitmint/dist/gitmint.browser.js"></script>
    {% else %}
        {% set CommentsClass = "Gitment" %}
        <link rel="stylesheet" href="https://imsun.github.io/gitment/style/default.css">
        <script src="https://imsun.github.io/gitment/dist/gitment.browser.js"></script>
    {% endif %}
```

分别查看两个`js`文件内的gitment授权实现：

[gitmint.browser.js](https://aimingoo.github.io/gitmint/dist/gitmint.browser.js)

```js
if (query.code) {
      var _oauth = this.oauth,
          client_id = _oauth.client_id,
          client_secret = _oauth.client_secret,
          proxy_gateway = _oauth.proxy_gateway;

      var code = query.code;
      delete query.code;
      var search = _utils.Query.stringify(query);
      var replacedUrl = '' + window.location.origin + window.location.pathname + search + window.location.hash;
      history.replaceState({}, '', replacedUrl);

      Object.assign(this, {
        id: replacedUrl,
        link: replacedUrl
      }, options);

      this.state.user.isLoggingIn = true;
      var logging = !proxy_gateway ? _utils.http.post('https://gh-oauth.imsun.net', { code: code, client_id: client_id, client_secret: client_secret }, '') : _utils.http.post('/login/oauth/access_token', 'code=' + code + '&client_id=' + client_id, proxy_gateway);
      logging.then(function (data) {
        _this.accessToken = data.access_token;
        _this.update();
      }).catch(function (e) {
        _this.state.user.isLoggingIn = false;
        alert(e);
      });
} else {
  this.update();
}
```

[gitment.browser.js](https://imsun.github.io/gitment/dist/gitment.browser.js)

```js
if (query.code) {
  var _oauth = this.oauth,
      client_id = _oauth.client_id,
      client_secret = _oauth.client_secret;

  var code = query.code;
  delete query.code;
  var search = _utils.Query.stringify(query);
  var replacedUrl = '' + window.location.origin + window.location.pathname + search + window.location.hash;
  history.replaceState({}, '', replacedUrl);

  Object.assign(this, {
    id: replacedUrl,
    link: replacedUrl
  }, options);

  this.state.user.isLoggingIn = true;
  _utils.http.post('https://gh-oauth.imsun.net', {
    code: code,
    client_id: client_id,
    client_secret: client_secret
  }, '').then(function (data) {
    _this.accessToken = data.access_token;
    _this.update();
  }).catch(function (e) {
    _this.state.user.isLoggingIn = false;
    alert(e);
  });
} else {
  this.update();
}
```

可以看到`gitmint.browser.js`多了代理的功能，如果没有配置代理地址`proxy_gateway`，二者均调用`https://gh-oauth.imsun.net`；
如果配置了那么`gitmint.broswer.js`会调用代理地址（[gitment](https://www.npmjs.com/package/gitment)内ajaxFactory函数会使用代理地址）。

对于[gitment模板](https://github.com/iissnan/hexo-theme-next/blob/master/layout/_third-party/comments/gitment.swig#L36):

```js
  function renderGitment(){
    var gitment = new {{CommentsClass}}({
        id: window.location.pathname, 
        owner: '{{ theme.gitment.github_user }}',
        repo: '{{ theme.gitment.github_repo }}',
        {% if theme.gitment.mint %}
        	lang: "{{ theme.gitment.language }}" || navigator.language || navigator.systemLanguage || navigator.userLanguage,
        {% endif %}
        oauth: {
        	{% if theme.gitment.mint and theme.gitment.redirect_protocol %}
        	    redirect_protocol: '{{ theme.gitment.redirect_protocol }}',
        	{% endif %}
        	{% if theme.gitment.mint and theme.gitment.proxy_gateway %}
        	    proxy_gateway: '{{ theme.gitment.proxy_gateway }}',
        	{% else %}
        	    client_secret: '{{ theme.gitment.client_secret }}',
        	{% endif %}
        	client_id: '{{ theme.gitment.client_id }}'
        }});
    gitment.render('gitment-container');
  }
```

可以看到，如果关闭了`mint`或者不配置`proxy_gateway`时，项目会用`client_secret`渲染`gitment`插件，调用`https://gh-oauth.imsun.net`，带有`client_id`，`client_secret`参数；
如果开启了`mint`并且配置了`proxy_gateway`，渲染`gitment`插件时不会传入`client_secret`，所以此时需要通过其他方式让代理服务调用github oauth接口时带有`client_secret`。

默认情况下，主题都是开启了`mint`(`mint`支持更丰富的gitment功能)，并且将`proxy_gateway`置为空。在地址`https://gh-oauth.imsun.net` SSL 证书失效时，浏览器报错。

## 原理分析

github oauth [获取access_token](https://developer.github.com/apps/building-oauth-apps/authorizing-oauth-apps/#2-users-are-redirected-back-to-your-site-by-github)的接口不支持CORS，那就需要一个支持CORS的服务进行代理，由该服务获取access_token，然后返回给浏览器。

CROS相关知识可以参考：阮一峰的日志[《跨域资源共享 CORS 详解》](http://www.ruanyifeng.com/blog/2016/04/cors.html)

## 解决办法

自己部署一个代理服务，代码见：[aishenghuomeidaoli/gh-oauth-server](https://github.com/aishenghuomeidaoli/gh-oauth-server)，已发起pull request。