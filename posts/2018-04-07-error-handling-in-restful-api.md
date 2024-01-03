---
title: RESTful API 中的错误处理
tags:
  - javascript
  - python
  - frontend
  - restful
categories: Programming
date: 2018-04-07 14:49:19
---


![RESTful API](/images/restful-api.png)

构建 Web 服务时，我们会使用 RESTful API 来实现组件间的通信，特别是在现今前后端分离的技术背景下。REST 是一种基于 HTTP 协议的通信方式，它简单、基于文本、且在各种语言、浏览器及客户端软件中能得到很好的支持。然而，REST 目前并没有一个普遍接受的标准，因此开发者需要自行决定 API 的设计，其中一项决策就是错误处理。比如我们是否应该使用 HTTP 状态码来标识错误？如何返回表单验证的结果等等。以下这篇文章是基于日常使用中的经验总结的一套错误处理流程，供读者们参考。

## 错误的分类

错误可以分为两种类型：全局错误和本地错误。全局错误包括：请求了一个不存在的 API、无权请求这个 API、数据库连接失败、或其他一些没有预期到的、会终止程序运行的服务端错误。这类错误应该由 Web 框架捕获，无需各个 API 处理。

本地错误则和 API 密切相关，例如表单验证、唯一性检查、或其他可预期的错误。我们需要编写特定代码来捕获这类错误，并抛出一个包含提示信息的全局异常，供 Web 框架捕获并返回给客户端。

例如，Flask 框架就提供了此类全局异常处理机制：

```python
class BadRequest(Exception):
    """将本地错误包装成一个异常实例供抛出"""
    def __init__(self, message, status=400, payload=None):
        self.message = message
        self.status = status
        self.payload = payload


@app.errorhandler(BadRequest)
def handle_bad_request(error):
    """捕获 BadRequest 全局异常，序列化为 JSON 并返回 HTTP 400"""
    payload = dict(error.payload or ())
    payload['status'] = error.status
    payload['message'] = error.message
    return jsonify(payload), 400


@app.route('/person', methods=['POST'])
def person_post():
    """创建用户的 API，成功则返回用户 ID"""
    if not request.form.get('username'):
        raise BadRequest('用户名不能为空', 40001, { 'ext': 1 })
    return jsonify(last_insert_id=1)
```

<!-- more -->

## 返回的错误内容

上例中，如果向 `/person` API 发送一个 `username` 为空的请求，会返回以下错误结果：

```text
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "status": 40001,
  "message": "用户名不能为空",
  "ext": 1
}
```

它包括以下几个部分：HTTP 状态码、自定义错误码、错误提示、以及额外信息。

### 正确使用 HTTP 状态码

HTTP 协议中预定义了丰富的状态码，其中 `4xx` 表示客户端造成的异常，`5xx` 表示服务端产生的异常。以下是我们在 API 中经常用到的几种状态码：

* `200` 响应结果正常；
* `400` 错误的请求，如用户提交了非法的数据；
* `401` 未授权的请求。在使用 `Flask-Login` 插件时，如果 API 的路由含有 `@login_required` 装饰器，当用户没有登录时就会返回这个错误码，而客户端通常会重定向到登录页面；
* `403` 禁止请求；
* `404` 请求的内容不存在；
* `500` 服务器内部错误，通常是未预期到的、不可恢复的服务端异常。

### 自定义错误码

客户端接收到异常后，可以选择弹出一个全局的错误提示，告知用户请求异常；或者在发起 API 请求的方法内部进行处理，如将表单验证的错误提示展示到各个控件之后。为了实现这一点，我们需要给错误进行编码，如 `400` 表示通用的全局错误，可直接弹框提示；`40001`、`40002` 则表示这类错误需要单独做处理。

```javascript
fetch().then(response => {
  if (response.status == 400) { // HTTP 状态码
    response.json().then(responseJson => {
      if (responseJson.status == 400) { // 自定义错误码
        // 全局错误处理
      } else if (responseJson.status == 40001) { // 自定义错误码
        // 自定义错误处理
      }
    })
  }
})
```

### 错误详情

有时我们会将表单内所有字段的验证错误信息一并返回给客户端，这时就可以使用 `payload` 机制：

```javascript
{
  "status": 40001,
  "message": "表单验证错误"
  "errors": [
    { "name": "username", "error": "用户名不能为空" },
    { "name": "password", "error": "密码不能少于 6 位" }
  ]
}
```

## Fetch API

对于 AJAX 请求，[Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) 已经逐渐成为业界标准。我们可以将其包装成一个方法，对请求结果进行错误处理。完整的代码可以在 GitHub （[链接](https://github.com/jizhang/blog-demo/blob/master/rest-error/src/request.js)）中查看。

```javascript
function request(url, args, form) {
  return fetch(url, config)
    .then(response => {
      if (response.ok) {
        return response.json()
      }

      if (response.status === 400) {
        return response.json()
          .then(responseJson => {
            if (responseJson.status === 400) {
              alert(responseJson.message) // 全局错误处理
            }
            // 抛出异常，让 Promise 下游的 "catch()" 方法进行捕获
            throw responseJson
          }, error => {
            throw new RequestError(400)
          })
      }

      // 处理预定义的 HTTP 错误码
      switch (response.status) {
        case 401:
          break // 重定向至登录页面
        default:
          alert('HTTP Status Code ' + response.status)
      }

      throw new RequestError(response.status)
    }, error => {
      alert(error.message)
      throw new RequestError(0, error.message)
    })
}
```

可以看到，异常发生后，该函数会拒绝（reject）这个 Promise，从而由调用方进一步判断 `status` 来决定处理方式。以下是使用 MobX + ReactJS 实现的自定义错误处理流程：

```javascript
// MobX Store
loginUser = flow(function* loginUser(form) {
  this.loading = true
  try {
    // yield 语句可能会抛出异常，即拒绝当前的 Promise
    this.userId = yield request('/login', null, form)
  } finally {
    this.loading = false
  }
})

// React Component
login = () => {
  userStore.loginUser(this.state.form)
    .catch(error => {
      if (error.status === 40001) {
        // 自定义错误处理
      }
    })
}
```

## 参考资料

* https://en.wikipedia.org/wiki/Representational_state_transfer
* https://alidg.me/blog/2016/9/24/rest-api-error-handling
* https://www.wptutor.io/web/js/generators-coroutines-async-javascript
