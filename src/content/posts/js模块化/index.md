---
title: js模块化
published: 2025-02-11
category: 技术人生
tags: ["js"]
image: js.png
---

# js 模块化

## 什么是模块化

首先可以通过黄玄的博客了解一下模块化的发展历程。
[JavaScript 模块化七日谈 - 黄玄的博客 | Hux Blog](https://huangxuan.me/2015/07/09/js-module-7day/)

## 模块化的标准

Commonjs：
Nodejs 是 commonjs 规范的主要实践者。
实际使用时用 **Module.exports**定义当前模块对外输出的接口，用 require 加载模块.
Common js 用同步的方式加载模块。在服务器端由于时硬盘架在，读取速度很快，所以不存在什么问题，如果是在浏览器端，受制于网络环境的影响，会导致页面卡顿假死的现象，更合理的方式就是使用异步加载的方式。

**amd 和 requirejs**

Amd（Asynchronous Module Definition）规范采用的是异步加载模块的方式，模块的加载不会影响后面语句的执行， 所有依赖于模块的语句都写在回调函数中，只有当模块全部加载完毕才会执行后面的回调函数。
Amd 是一种规范，异步加载模块的一种思想，而 requirejs 是 amd 规范的一种应用。

requirejs 的基本使用[Javascript 模块化编程（三）：require.js 的用法 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2012/11/require_js.html)

其大概思想就是：
引入 requirejs 文件，设置为异步加载，在标签中属性声明网页程序的主模块。
在主模块中声明要引入的基础模块。
使用 requeire.config（）指定引用路径，define（）定义模块，require（）加载模块。

```javascript
/** 网页中引入require.js及main.js **/
<script src="js/require.js" data-main="js/main"></script>;

/** main.js 入口文件/主模块 **/
// 首先用config()指定各模块路径和引用名
require.config({
  baseUrl: "js/lib",
  paths: {
    jquery: "jquery.min", //实际路径为js/lib/jquery.min.js
    underscore: "underscore.min",
  },
});
// 执行基本操作
require(["jquery", "underscore"], function ($, _) {
  // some code here
});
```

```javascript
// 定义math.js模块
define(function () {
  var basicNum = 0;
  var add = function (x, y) {
    return x + y;
  };
  return {
    add: add,
    basicNum: basicNum,
  };
});
// 定义一个依赖underscore.js的模块
define(["underscore"], function (_) {
  var classify = function (list) {
    _.countBy(list, function (num) {
      return num > 30 ? "old" : "young";
    });
  };
  return {
    classify: classify,
  };
});

// 引用模块，将模块放在[]内
require(["jquery", "math"], function ($, math) {
  var sum = math.add(10, 20);
  $("#sum").html(sum);
});
```

## cmd 和 seajs

Cmd（Common Module Definition）模块加载规范。
Cmd 和 amd 规范差不多，但是 cmd 提倡的是依赖就近，延迟执行。而 amd 提倡的是依赖提前，提前执行。
Seajs 就是实现 cmd 规范的一个应用。

```javascript
/** AMD写法 **/
define(["a", "b", "c", "d", "e", "f"], function (a, b, c, d, e, f) {
  // 等于在最前面声明并初始化了要用到的所有模块
  a.doSomething();
  if (false) {
    // 即便没用到某个模块 b，但 b 还是提前执行了
    b.doSomething();
  }
});

/** CMD写法 **/
define(function (require, exports, module) {
  var a = require("./a"); //在需要时申明
  a.doSomething();
  if (false) {
    var b = require("./b");
    b.doSomething();
  }
});

/** sea.js **/
// 定义模块 math.js
define(function (require, exports, module) {
  var $ = require("jquery.js");
  var add = function (a, b) {
    return a + b;
  };
  exports.add = add;
});
// 加载模块
seajs.use(["math.js"], function (math) {
  var sum = math.add(1 + 2);
});
```

ES6 module
Js 从语言层面上规定了模块化的标准，旨在通过标准来实现浏览器和服务器环境上模块化的统一解决方案。

Es6 加载模块是在编译时就加载好的，而不是运行时，所以无法时间条件加载。正因为如此，才使得静态分析成为可能。

ES6 模块和 common js 的三大差异

1. common JS 模块的输出的是一个值的拷贝，es 6 模块的输出是值的引用。
2. common js 模块是运行时加载， es6 模块是编译时输出接口。
3. Common js 模块的 require（）是同步加载模块，es6 模块的 import 命令是异步加载，有一个独立的模块依赖的解析阶段。

Es6 的模块导出类似于 unix 中 符号连接。软链接。

原生 js 组织阶段 - 在线处理阶段 - 预处理阶段

1. 原生阶段使用原生的 js 去组织模块
2. 在线处理使用的 amd 或者 cmd 或者 es6+babel 这种去处理模块间的依赖，最终是在浏览器去处理依赖关系。
3. 预处理阶段使用 broswerify 或者 webpack 这种预处理工具将模块之间打包成单个或者几个问题，压缩首次访问页面 http 的请求数量，提高性能。

预处理阶段会碰到的问题是如果处理单个文件过大的问题。
可以通过代码拆分的插件来阶段将实现两个大的功能点：

1. 实现第三方库和业务代码的分离。
2. 实现按需加载
