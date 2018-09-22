---
layout:     post
title:      "关于RESTful API的一些理解"
date:       2018-09-22 00:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-09-22-01-bg.jpg"
tags:
    - RESTful
---

## 前言

随着互联网的发展，各种技术的层出不穷，与服务端进行通信的不再是简单的浏览器、手机，更有平板、各种云服务购买方、其他专有设备....

因此，在设计服务端API时，就需要一个合理的规范机制，方便不同的调用方与服务端进行通信。RESTful API是目前比较成熟的一套互联网应用程序的API设计理论，主要优点是规范、易懂、优雅，对设计应用API具有一定的指导意义。

本文是站在自己的角度去理解设计RESTful API的一些心得，其中某些地方可能与标准有一定的出入。

## 协议

与API的通信协议，尽量使用HTTPS

## 域名

尽量将API部署在专用域名之下

`https://api.example.com`

## 版本

版本号拼接在URL或者放在header中都可以。例如：

`https://api.example.com/v1/`

或

```
https://api.example.com
version=v1
```

个人更倾向于使用第一种方式，简单、直观

## 资源

[RFC3689](https://tools.ietf.org/html/rfc2396)中定义了URI通用语法，不过在RESTful架构中，我们更关注于定义中的path部分，主要在于它明确了具体的操作实体——资源。

在RESTful架构中，每个网址代表一个资源，所以网址中不能有动词，只能有名词，而且所用的名词往往都是复数，这主要归根于所操作的资源一般都对应于数据库中的某个表，表是同种类型的集合，所以API中的名词常使用复数。

举例来说，有一个API提供动物园的信息，还包括各种动物和雇员的信息，则它的路径应该设计成下面这样。
```
https://api.example.com/v1/zoos
https://api.example.com/v1/animals
https://api.example.com/v1/employees
```

## HTTP动词

对于资源的具体操作，由HTTP动词表示，资源自身只具备描述性。

一些常用的HTTP动词：

- GET: 获取资源
- POST: 创建资源
- PUT: 更新资源(客户端提供改变后的完整资源)
- PATCH: 更新资源(客户端提供改变的属性)
- DELETE: 删除资源
- HEAD: 获取资源的HEADER信息
- OPTIONS: 获取资源提供的动词

在实际的开发中，对资源的操作一般限定在CRUD上，其对应的HTTP动词分别为POST、GET、PUT、DELETE，较少使用PATCH来更新资源，下面是一些简单使用例子:
```
GET /zoos：列出所有动物园
POST /zoos：新建一个动物园
GET /zoos/ID：获取某个指定动物园的信息
PUT /zoos/ID：更新某个指定动物园的信息
DELETE /zoos/ID：删除某个动物园
```

## 过滤

如果记录数量很多，服务器不可能都将它们返回给用户。API应该提供参数，过滤返回结果。

下面是一些常见的参数：

- ?limit=10：指定返回记录的数量
- ?offset=10：指定返回记录的开始位置。
- ?page=2&per_page=100：指定第几页，以及每页的记录数。
- ?sortby=name&order=asc：指定返回结果按照哪个属性排序，以及排序顺序。
- ?animal_type_id=1：指定筛选条件

这里提一下过滤中关于分页的设计，当前分页布局一般分为两种，一种是在Web端比较常见的有底部分页栏的电梯式分页，另外一种是在APP通过上拉、下拉方式加载数据的流式分页。那这两种分页的API应该怎么设计呢？

电梯式的分页需要提供page（页数）和 per_page（每页数量)，例如：
```
/v1/zoos?page=1&per_page=10
```
服务端的返回需要额外返回total（总记录数）、可选的当前页、每页的数量、总页数、是否有上一页、是否有下一页之类的信息，而这些信息都可以通过total、per_page、page计算得出。

客户端的设计同样可以采用上述思想，并且不需要查询总数(好处是少了一次数据库的查询，坏处是客户端无法知道是否还有数据从而多拉一次)。另外，电梯式的分页始终避免不了数据重复的问题，所以又诞生了另外一种分页方式——游标式分页

对Redis比较熟悉的读者应该有使用过scan命令，游标式的分页思想与其类似。假定客户端的feed流以id降序排列，那游标式分页的请求应该是这个样子的：
```
/zoos?cursor=xxx&per_page=10
```
初次请求时，cursor为0，后续的请求cursor值从最近的返回结果中获取，服务端对应的返回结果为：
```
{
    "pagination": {
        "cursor": 1000,
        "per_page": 10
    },
    "data": [...]
}
```
当服务端返回cursor为0时，表明没有多余的数据了。关于服务端怎么计算cursor的逻辑也很简单，从数据库中拉数据时，拉取per_page + 1，cursor的值就取自第per_page + 1条数据的主键。

游标式分页不会出现电梯式分页中的数据重复问题，但它也存在一些限制，使用时需要看对应的场景。

## 限流

在某些特殊的场景下，我们需要对API进行一些访问频率限制，关于访问频率的控制信息，可通过设置如下所示的Header:

| Header Name | Description |
| :------: | :------: |
| X-RateLimit-Limit	| 每小时允许的最大访问次数(时间窗口可自行设置) |
| X-RateLimit-Remaining	| 当前频率限制窗口所剩的次数 |
| X-RateLimit-Reset	| 距离当前频率限制窗口重置的秒数(UTC epoch seconds) | 

关于[Github V3](https://developer.github.com/v3/)采用的限流控制信息与此一致，直观感受下：
```
curl -i https://api.github.com/users/octocat
HTTP/1.1 200 OK
Date: Mon, 01 Jul 2013 17:27:06 GMT
Status: 200 OK
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 56
X-RateLimit-Reset: 1372700873
```
将限流控制信息告知调用方的好处在于，调用方可根据限流控制信息来控制自身访问频率，避免了超出频率限制而无效的请求流量

## 响应

HTTP状态码从逻辑上大致分为5类:

- 1xx: 信息提示
- 2xx: 成功
- 3xx: 重定向
- 4xx: 客户端错误
- 5xx: 服务端错误

尽量使用HTTP状态码表示请求的处理结果，如果遇上HTTP状态码无法明确的表达错误信息时，可以在返回结果中包一层自定义的返回码，例如:
```
{
    code: 1001
    msg: "用户名或者密码错误"
}
```
下面是一些常见的HTTP状态码:

- 200 OK - [GET]：服务器成功返回用户请求的数据，该操作是幂等的（Idempotent）。
- 201 CREATED - [POST/PUT/PATCH]：用户新建或修改数据成功。
- 202 Accepted - [*]：表示一个请求已经进入后台排队（异步任务）
- 204 NO CONTENT - [DELETE]：用户删除数据成功。
- 400 INVALID REQUEST - [POST/PUT/PATCH]：用户发出的请求有错误，服务器没有进行新建或修改数据的操作，该操作是幂等的。
- 301 Moved Permanently - [*]：永久重定向
- 302 Found - [*]：临时重定向
- 401 Unauthorized - [*]：表示用户没有权限（令牌、用户名、密码错误）。
- 403 Forbidden - [*] 表示用户得到授权（与401错误相对），但是访问是被禁止的。
- 404 NOT FOUND - [*]：用户发出的请求针对的是不存在的记录，服务器没有进行操作，该操作是幂等的。
- 406 Not Acceptable - [GET]：用户请求的格式不可得（比如用户请求JSON格式，但是只有XML格式）。
- 410 Gone -[GET]：用户请求的资源被永久删除，且不会再得到的。
- 422 Unprocesable entity - [POST/PUT/PATCH] 当创建一个对象时，发生一个验证错误。
- 500 INTERNAL SERVER ERROR - [*]：服务器发生错误，用户将无法判断发出的请求是否成功。


## 参考资料

- [RESTful API设计指南](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)
- [REST API V3](https://developer.github.com/v3/)
- [我所认为的RESTful API最佳实践](http://www.scienjus.com/my-restful-api-best-practices/)