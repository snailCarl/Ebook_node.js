# 回顾和简介

## Web服务架构图

<script>
  mermaidAPI.initialize({
    // theme: mermaid.darkTheme,
    flowchart:{
      useMaxWidth: true,
      htmlLabels: true
    }
  })

  // val callback = function(){
  //   console.log('A callback was triggered');
  // }
</script>

```mermaid
graph LR
  Request["Webpage<br>App<br>Webserve"]-- request -->Server["HttpServer<br>(http.createServer)"]
  Server-- response -->Request
  <!-- click Request callback "Tooltip for a callback" -->
  <!-- click Server "http://www.github.com" "This is a link" -->
```

## 知识点总结

1. 创建http服务(createServer方法)并绑定端口(listen方法)，监听httpServer事件(on方法)，常见事件：'request'
1. 两个EventEmitter实例，request 和 response，分别封装了用户请求和服务器响应。监听EventEmitter事件(on方法)；常见事件：'data'，'end'，'error'等。
1. 提取request数据；response塞入数据返回给请求者。
1. 新的语法知识：函数参数，不同于其他语言，javascript可以直接将一个函数作为参数；掌握如何定义函数参数。
1. 了解异步处理机制：回调(即传入的函数参数)会被Node.js放到相关处理队列中，并在后续的时间周期中被处理。

后面的Node.js内部的时间循环机制以及Timer等可在后面的项目中逐渐深入了解和使用。

## 中间件简介

上面我们了解了如何在Node.js中启动http服务，并相应请求返回处理结果。但是，实际项目中很少直接从http开始搭建服务。相反我们会使用一些成熟的第三方库来构建自己的http服务。这些库内部的实现还是上述知识点，不过他们封装了很多好用的API，减少了重复的基础构建工作，或者拓展了相关功能。稍后，我们将重点介绍几个常用的第三方库：'express', 'body-parser', 'xml2js', 'request'这些库都可以在[npm仓库](https://www.npmjs.com)上找到介绍和使用方法。

1. [express](https://www.npmjs.com/package/express): 封装了httpserve，利用其api可以方便的创建http服务，并设置路由。
1. [body-parse](https://www.npmjs.com/package/@types/body-parser): 对用户request的报文进行解析，生成JSON格式的数据对象。
1. [xml2js](https://www.npmjs.com/package/xml2js): 将xml格式的数据解析成JSON格式的数据对象。
1. [request](https://www.npmjs.com/package/xml2js): 封装了Node.js作为用户端发向其他Web服务的请求接口（注意跟EventEmitter的request区分），这个是用来向别人发请求，并且接收对方的response的。

## 项目配置

开始行走江湖前，先了解一下脚手架配置，我们从无到有来创建一个项目。

1. 创建项目目录并切换到该目录

  ```bash
  $ mkdir prjdemo && cd prjdemo
  ```
1. 生成配置文件`package.json`

  ```bash
  $ npm -y init
  ```
1. 安装插件(比如express)

  ```bash
  $ npm i express
  ```

此时就可以在项目目录下编写代码文件了，并在代码中引用需要的已经安装好的第三方包。npm还有其他很多相关指令，自行了解。这里对于安装指令(i 或者 install)稍微说明一下，安装分两种：

1. 全局安装：安装到本地全局库目录下，一次安装后续所有项目都可以无需再安装。

  ```bash
  $ npm i express -g
  ```

1. 本项目安装，并写入项目配置文件(package.json)，推荐方式。

  ```bash
  $ npm i express --save
  ```

对于写好了package.json的项目，在共享项目时，只需执行`npm i`就会自动在项目目录下安装好package.json中记录的依赖包。package.json的其他配置项目，自行了解一下。