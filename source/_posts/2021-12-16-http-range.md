---
title: HTTP Range - 分段下载
date: 2021-12-16 12:00:00 +0800
tags: [http, range, content-range]
---

# request header

```
Range: <unit>=<range-start>-
Range: <unit>=<range-start>-<range-end>
Range: <unit>=<range-start>-<range-end>, <range-start>-<range-end>, ...
```

* `<unit>`：范围的单位，通常是字节（bytes）
* `<range-start>`：范围的起始值
* `<range-end>`：范围的结束值。这个值是可选的，如果不存在表示此范围一直延伸到文档结束

`Range` 是一个请求首部，值是一个或多个 `范围`，告知服务器返回文件的哪一部分

1. 如果是请求单个范围，服务器返回 `206 Partial Content`
2. 如果请求多个范围，服务器会以 multipart 文件的形式将其返回
3. 如果请求的范围不合法，服务器返回 `416 Range Not Satisfiable`
4. 服务器允许忽略 Range 首部从而返回整个文件（比如服务器不支持分段下载），状态码用 200

# response header (206)

`206 Partial Content` 表示请求已成功，并且主体包含所请求的数据区间，该数据区间是在请求的 Range 首部指定的

1. 如果只包含一个数据区间，那么整个响应的 `Content-Type` 首部的值为所请求的文件的类型，同时包含  `Content-Range` 首部
2. 如果包含多个数据区间，那么整个响应的 `Content-Type` 首部的值为 `multipart/byteranges`，其中一个片段对应一个数据区间，并提供 `Content-Range` 和 `Content-Type` 描述信息

```
Content-Range: <unit> <range-start>-<range-end>/<size>
Content-Range: <unit> <range-start>-<range-end>/*
Content-Range: <unit> */<size>
```

* `<size>`：整个文件的大小，如果大小未知则用 `*` 表示

# 应用

多线程下载、分布式下载

1. 发送 `Head` 请求确定服务端是否支持 `Range` 以及确定文件 `Content-Length`
2. 划分不同的下载任务
    * 任务一： Range: bytes=0-99
    * 任务二： Range: bytes=100-199
    * 任务三： Range: bytes=200-330
    * ...
3. 分发不同的下载任务给不同的节点或线程
4. 合并下载结果

断点续传：记录文件长度 `Content-Length` 和已下载的数据量，中断下载后重新开始时，从上一次下载结束的偏移量开始

# 阿里云 OSS 分段下载的例子

通过 HTTP Range 获取大文件的部分内容示例如下

```
Get /ObjectName HTTP/1.1
Host:xxxx.oss-cn-hangzhou.aliyuncs.com
Date:Tue, 17 Nov 2015 17:27:45 GMT
Authorization:SignatureValue
Range:bytes=[$ByteRange]
```

* `[$ByteRange]` 指请求资源的范围，单位为字节，有效区间在 0 至 content-length - 1 的范围内
* `Range: bytes=0-499` 表示第 0 ~ 499 字节范围的内容
* `Range: bytes=500-999` 表示第 500 ~ 999 字节范围的内容
* `Range: bytes=-500` 表示最后 500 字节的内容
* `Range: bytes=500-` 表示从第 500 字节开始到文件结束部分的内容
* `Range: bytes=0-` 表示第一个字节到最后一个字节，即完整的文件内容
* OSS 不支持多 `Range` 参数，即不支持指定多个范围。如果指定多个范围 OSS 只返回第一个 `Range` 的数据，例如 `Range:bytes=0-499,500-999` OSS 只返回 0 - 499 字节范围的内容

如果 HTTP Range 请求合法，响应 `206` 并在响应头中包含 `Content-Range`；如果 HTTP Range 请求不合法或者指定范围不在有效区间，会导致 `Range` 不生效，响应 200 并传送整个 Object 内容

```
# 请求 Object 资源 0-499 字节范围内的内容

GET /ObjectName
Range: bytes=0-499
Host: bucket.oss-cn-hangzhou.aliyuncs.com
Date: Fri, 18 Oct 2019 02:51:30 GMT
Authorization: Sigature

206 (Partial Content)
content-length: 500
content-range: bytes 0-499/1000
connection: keep-alive
etag: "CACF99600561A31D494569C979E6FB81"
x-oss-request-id: 5DA928B227D52731327DE078
date: Fri, 18 Oct 2019 02:51:30 GMT
[500 bytes of object data]
```

```
# 指定范围超出有效区间，导致 Range 不生效，响应 200 并传送整个 Object 内容

GET /ObjectName
Range: bytes=1000-2000
Host: bucket.oss-cn-hangzhou.aliyuncs.com
Date: Fri, 18 Oct 2019 02:56:24 GMT
Authorization: Sigature

200 (OK)
content-length: 1000
etag: "CACF99600561A31D494569C979E6FB81"
x-oss-request-id: 5DA929D9CCCC823835CBE134
date: Fri, 18 Oct 2019 02:56:25 GMT
[1000 bytes of object data]
```

# 参考

* [206 Partial Content](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/206)
* [Range](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Range)
* [Content-Range](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Range)
* [天天见之HTTP Header Range 和 Content-Range 你真的了解吗？](https://zhuanlan.zhihu.com/p/126991550)
* [如何通过HTTP Range请求分段获取OSS资源](https://help.aliyun.com/document_detail/39571.html)