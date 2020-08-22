---
title: Http 基础
date: 2020-08-11 19:48:46
category: 计算机网络
---

## 前言

本文主要记录下 HTTP 的一些基础知识，包括请求与响应以及常见的请求方法、状态码以及请求响应头。 <!-- more -->

## 浏览器内核

其实就是浏览器的渲染引擎，用于渲染 HTML。常见的有 WebKit。

## Http 链接

分为三个部分，Scheme(协议)、Address(地址)、Path(路径)。

## Http 请求

分为三个部分，请求行、请求头、请求体。

### 请求行

分为 Method(请求方法)、Path(路径)、HttpVersion(版本)。例: GET /users HTTP/1.1

#### Http 版本

0.9、1.0 已经被废弃，最新的是 Http 2.0，但是主流使用的都是 1.1 因为协议升级需浏览器和服务端都升级，但是在 **Android 开发中可以和服务端约定使用 Http 2.0**，因为双端都是同一个公司的。

### 请求头

在请求行后另起一行(无空行)以键值对的形式书写，可以有若干个·。

### 请求体

在请求行后**空一行**书写，可能为普通的文本信息，也可能为图片等二进制数据，常用于 POST、PUT 请求。

## Http 响应

分为三个部分，状态行、响应头、响应体。

### 状态行

分为 HttpVersion(版本)、StatusCode(状态码)、StatusMessage(状态码描述)。例: HTTP/1.1 200 OK

### 响应头

在状态栏后另起一行(无空行)以键值对的形式书写，可以有若干个·。

### 响应体

在响应行后**空一行**书写，可能为普通的文本信息，也可能为图片等二进制数据。

## 请求方法

请求方法主要关注 GET、POST、PUT、DELETE、HEAD 这 5 种方法。

### GET

无请求体，用于获取资源。幂等性。

### POST

有请求体，用于新增、修改资源。无幂等性。

### PUT

有请求体，用于修改资源。幂等性。注：与 POST 的区别只是其只用于修改资源。

### DELETE

无请求体，用于删除资源。幂等性。如：DELETE /users/1 HTTP/1.1。

### HEAD

无**响应体**，用于获取信息。幂等性。常见应用场景为下载文件时先使用 HEAD 请求方法获取待下载文件的信息(从 Header 中可以获取到)，接着再使用 GET 请求去下载，这么做的优势是 HEAD 请求会比 GET 请求快。

### 小结

以上 5 种请求方式中，GET、DELETE 无请求体，HEAD 无响应体。

## 状态码

状态码主要用于对请求结果进行分类，共分为 1xx、2xx、3xx、4xx、5xx。

### 1xx

临时性消息，常见有 100 继续发、101 协议切换。

#### 100

如果请求体内容很大，需要分成好几个请求，那么可以添加请求头 Expect: 100-continue ，服务端会返回状态码 100，让你继续发，直到没有 Expect 请求头，服务端再做一次响应。

#### 101

协议切换，比如你想切换成 Http 2.0，就得先用 Http 1.1 发送一个请求，带上请求头 Upgrade: h2c ，如果服务端支持那么返回 101，后续就可以使用 Http 2.0 进行通信，再比如你想和服务端建立 WebSocket 连接，首先得先将 协议从 Ws 切换为 Http，发送一个 Get 请求，带上请求头 Upgrade: webSocket ，如果服务端支持那么返回 101。

### 2xx

成功，常见有 200 成功、201 创建成功、206 部分数据请求成功，用于断点下载。

### 3xx

重定向，常见有 301 永久移动、302 临时移动、304 文件未改变。 如 Http 请求百度返回 301，读取 Location 响应头指定地址再次请求。

### 4xx

客户端(发起请求方)错误，常见有 401 未授权，可能没登录、404 资源找不到。

### 5xx

服务端错误，常见有 500、502，时机为服务端抛出异常。

## 请求响应头

Http 消息的元数据（消息的消息）。

### Host

存放目标主机名，作用是给哪些拥有多个子主机的主机看的，让其区分具体请求的是哪个子主机。

如一台 IP 地址为 61.135.169.125 的主机虚拟为三台主机分别部署谷歌、百度、淘宝，请求到达后该主机就通过 Host 请求头来选择目标子主机。

注意：

1. 子主机要和同一主机不同端口区分，一台主机的子主机完全可以有相同端口号。
2. 在发起真正的 Http 请求前，客户端就会去访问 DNS 服务器获取目标主机的 IP 地址。因此该请求头**不是用于网络寻址的**。

### Content-Type

表示请求或响应体数据的类型，有以下几个可选值：

1. text/html，网页 **默认**。

2. text/xml，XML。

3. x-www-form-urlencoded， 普通表单，请求体内容就跟 GET 请求后参数格式一样。

4. multipart/form-data， 多块表单，如注册既有邮箱、昵称等文本信息，也有头像等文件。

    ```http
    POST /users HTTP/1.1
    Host: jiaoyihu-test.com
    Content-Type: multipart/form-data; boundary=----
    WebKitFormBoundary7MA4YWxkTrZu0gW
    Content-Length: 2382
    ------WebKitFormBoundary7MA4YWxkTrZu0gW
    Content-Disposition: form-data; name="name"
    rengwuxian
    ------WebKitFormBoundary7MA4YWxkTrZu0gW
    Content-Disposition: form-data; name="avatar";
    filename="avatar.jpg"
    Content-Type: image/jpeg
    JFIFHHvOwX9jximQrWa......
    ------WebKitFormBoundary7MA4YWxkTrZu0gW--
    ```

    使用 boundary 区分每一块，然后每一块内部又有对应的字段名，完全可以上传一些文本信息，若干个文件。

5. application/json， JSON。注意：retrofit 参数只需 @Body body: RequestBody，然后创建一个 content-type 为 application/json 类型的 RequestBody 实例传入即可。

6. image/jpeg，Jpg 格式图片，可用于上传或下载，使用方式同上，只是把 content-type 改成 image/jpeg。

### Content-Length

表示请求或者响应体所包含的字节数，接收方会在头后截取该长度的数据当做请求或响应体，如果没有该头，由于传输的数据可能为图片等二进制数据，会导致接收方无法判断哪里截断。

### Transfer-Encoding

一般取值为 chunked ，表示分块传输，有该请求头可以不需要 content-length 。如服务端没法短时间将所有数据查询出来，则可以添加该请求头分块传输，来提高网络性能。

```http
HTTP/1.1 200 OK
Content-Type: text/html
Transfer-Encoding: chunked

4
Tran
3
fer
0
```

最后的 0 表示响应结束。

### Location

指定重定向的 Url。注意：如果使用 OkHttp 访问 `Http://www.baidu.com` 是可以获取到百度的网页数据，但是其实 OkHttp 发起了两个请求，首次请求 `Http://www.baidu.com`，服务端返回状态码 301 ，响应头 Location: `Https://www.baidu.com` OkHttp 内部的 BridgeInterceptor 执行重定向再次访问 `Https://www.baidu.com` 。

### User-Agent

用户代理，标识了请求方的身份。服务端可以使用该字段来区分是手机还是电脑浏览器请求的，然后返回对应的网页内容。很多手机浏览器都支持桌面模式原理就是把该请求头内容修改了。

### Accept-Ranges

如果有该响应头，那么表示服务端支持访问部分数据。

### Range

当响应头包含 Accept-Ranges 后才能使用，指定访问的部分数据，常用于断点续传，多线程下载。

### Accept

客户端能接受的数据类型。如 text/html。

### Accept-Charset

客户端接受的字符集。如 utf-8。

### Accept-Encoding

客户端接受的压缩编码类型。如 gzip。

### Content-Encoding

指定请求响应体的压缩类型。如 gzip。

### Cache-Control

缓存控制，暂时只需了解该请求头处理缓存优先级最高。

### Last-Modified

最近更改时间，对应请求头 If-Modified-Since 。

### Etag

资源标签，对应请求头 If-none-match。

## REST API

REST 是一种软件架构风格，一般而言满足需要满足以下两点：

1. 提供一个页面展示所有的 API，如 api.github.com。
2. 正确使用 Http，如删除使用 DELETE、获取资源使用 GET。