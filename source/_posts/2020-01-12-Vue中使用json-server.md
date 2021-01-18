---
title: Vue中使用json-server
date: 2020-01-12 12:19:17
categories:
- Json-server
tags:
- json-server
- Vue
---

<center><font size=4 color="red">Vue中使用json-server</font></center>

<!--more-->

## Vue中使用json-server

#### 安装json-server

全局安装

```powershell
npm install -g json-server
```

局部安装

```powershell
npm install --save json-server
```

####  **提供json数据文件** 

 在项目根目录下，新建一个 JSON 文件db.json 

```json
{
    "student": [
      {"id":1, "name":"zhangsan"},
      {"id":2, "name":"lisi"},
      {"id":3, "name":"wangwu"}
    ]
}
```

####  **配置json-server** 

 在build\webpack.dev.conf.js下配置

```js
/*----------------jsonServer---------*/
/*引入json-server*/
const jsonServer = require('json-server')
/*搭建一个server*/
const apiServer = jsonServer.create()
/*将db.json关联到server*/
const apiRouter = jsonServer.router('db.json')
const middlewares = jsonServer.defaults()
apiServer.use(middlewares)
apiServer.use(apiRouter)
/*监听端口*/
apiServer.listen(3000, () => {
  console.log('JSON Server is running')
})
```

####  **访问数据** 

 配置完成后，要npm dev run 重启项目，然后再地址栏输入http://localhost:3000 就可以访问到数据。

####  **设置代理** 

 最后做一下浏览器代理设置，因为json-server的访问端口是3000，Vue的访问端口是8000，为了使用json-server时端口也用8000，所以设置一下代理。在 config/index.js中 

```js
/*代理配置表，在这里可以配置特定的请求代理到对应的API接口*/
/* 下面的例子将代理请求 /api/student  到 http://localhost:3000/student*/
proxyTable: {
  '/api': {
    changeOrigin: true,// 如果接口跨域，需要进行这个参数配置
    target: 'http://localhost:3000',// 接口的域名
    pathRewrite: {
      '^/api': ''//后面可以使重写的新路径，一般不做更改
    }
  }
}
```

####  **验证代理是否成功**

 在浏览器输入地址：http://localhost:8080/api/

![](jsonServer.png)

#### Vue中 **使用** 

 使用vue-resouce发送Ajax获取数据 

```js
this.$http.get('/api/student')//代替http://localhost:3000/student
  .then((res) => {
    this.newsList = res.data
  }, (err) => {
    console.log(err)
  })
```
