---
title: json-server
date: 2020-01-12 12:17:06
categories:
- Json-server
tags:
- json-server
---

<center><font size=4 color="red">json-server入门</font></center>

<!--more-->

## json-server

#### http-server

用于模拟http请求，安装

```powershell
$ npm i http-server -g
```

查看是否安装完成。其中的`hs`是`http-server`的简写

```powershell
$ hs -V
Starting up http-server, serving ./
Available on:
  http://192.168.31.235:8080
  http://192.168.120.1:8080
  http://88.88.88.1:8080
  http://192.168.25.1:8080
  http://127.0.0.1:8080
Hit CTRL-C to stop the server
```

启动http-server。其中的`-o`意思是启动时打开浏览器

```powershell
$ hs -o
Starting up http-server, serving ./
Available on:
  http://192.168.31.235:8080
  http://192.168.120.1:8080
  http://88.88.88.1:8080
  http://192.168.25.1:8080
  http://127.0.0.1:8080
Hit CTRL-C to stop the server
```

#### RESTful API

* 面向资源编程
* 地址即资源
* 所有的东西都是资源，所有操作都通过对资源的增删改查实现
* 对资源的增删改查对应URL的操作分别是POST,DELETE,PUT,GET
* 如果要测试POST,DELETE,PUT，使用postman即可，浏览器只能查询

```
http://localhost:8080/users          -> 所有的用户数据
http://localhost:8080/products       -> 所有的产品数据
http://localhost:8080/users/1/name   -> 编号为1的用户的名称
http://localhost:8080/products/iphone-> 特指iphone这个商品
```



#### 第三方模拟数据工具

* mock.js:无法持久化数据。 http://mockjs.com/ 

  * mock的使用文档： http://mockjs.com/examples.html 
  * mock拦截请求，返回假数据的方法

  ```js
  //1.引入mock.js文件
  //2.使用Mock.mock()
  //3个参数。'/users'：拦截http://localhost:8080/users的请求；'get'：拦截的请求方式是get方式；{hello:'mock.js'}：返回的假数据 
  Mock.mock('/users','get',{
      hello:'mock.js'
  })
  ```

* json-server 

#### json-server

安装json-server

```powershell
$ npm install -g json-server
```

启动json-server。其中的`db.json`是数据文件

```powershell
$ json-server --watch db.json
```

#### json-server基本特性

具体的基本特性使用方式参考：https://github.com/typicode/json-server

db.json测试数据库，访问地址： http://localhost:3000/student 

```json
{
    "student": [
      {"id":1, "name":"zhangsan"},
      {"id":2, "name":"lisi"},
      {"id":3, "name":"wangwu"}
    ]
}
```

* 标准的RESTful API

* 支持过滤，访问地址： http://localhost:3000/student?q=zhangsan

  ```json
  [
    {
      "id": 1,
      "name": "zhangsan"
    }
  ]
  ```

* 支持分页,访问地址：http://localhost:3000/student?_limit=2&_page=1。意思是访问第一页page，该页显示两条数据

  ```json
  [
    {
      "id": 1,
      "name": "zhangsan"
    },
    {
      "id": 2,
      "name": "lisi"
    }
  ]
  ```

* 支持排序

* 支持全文检索

* 支持关系

* 支持数据分割

* 支持操作符（大于小于）

* 支持JSONP

* 支持CORS
