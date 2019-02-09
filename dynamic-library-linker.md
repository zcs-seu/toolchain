# 要解决的问题

* 在不共享源代码的前提下
* 复用其他人的工作
* 不共享源代码可能是避免版权被盗
* 不共享源代码也可能是因为源代码的种类繁多，运行时库更方便统一

# 解决方案

exectuable不会携带所有的算法实现。它会构建在executor提供的builtin api的基础上。同时也会利用第三方的 dynamic library 依赖 executor 的 dynamic linker 动态链接进来使用。这个动态链接其他人的工作的机制在很多种机器中都有提供。

## 构成

* executable：动态链接发起的源头，executor第一个加载的东西
* dynamic library：提供被复用的算法，动态链接库
* dynamic linker：executor的一个组件，提供动态链接的能力

## 衍生的问题

* [如何标识并定位运行时库](dynamic-library-resolver.md)
* [如何引用运行时库指定的接口](symbol-binder.md)

# 解决方案案例

## 传统浏览器

* executable：直接嵌入到 html 的 `<script>` 标签的 JavaScript 代码
* dynamic library：由 html 文件的 `<script src="xxx">` 引用的 .js 文件
* dynamic linker：浏览器自身，通过全局变量实现彼此的互通。script标签的顺序决定了加载顺序，后被加载的js可以引用前面js定义的全局变量。

```js
// http://localhost/library.js
function call_library() { 
  console.log('i am the library')
}
```

```html
// http://localhost/index.html
<html>
<head>
<script src="./library.js"></script>
<script>
console.log('i am the executable')
call_library() // window.call_library()
</script>
</head>
<body>
</body>
</html>
```

executable 和 dynamic library 之间的通过全局变量 window 上的全局变量实现互相调用。
`call_library` 这个函数其实就是定义在 `window` 上的变量。
用浏览器访问 `http://localhost/index.html` 在浏览器的控制台输出

```
i am the executable
i am the library
```

JavaScript 代码内没有直接的动态加载的支持，用 script 标签加载的 url 无法动态计算出来。一个hack的方法是通过 DOM API 创建 script 标签。

```js
// http://localhost/library.js
console.log('i am the library')
```

```html
// http://localhost/index.html
<html>
<head>
<script>
var head= document.getElementsByTagName('head')[0];
var script= document.createElement('script');
script.type= 'text/javascript';
script.src= './library.js';
head.appendChild(script);
</script>
</head>
<body>
</body>
</html>
```

## 支持 ES6 Module 的浏览器

支持 ES6 Module 的浏览器，Chrome 从61起

* exectuable：直接嵌入到 html 的 `<script type="module">` 标签的 JavaScript 代码
* dynamic library：用 url 可以访问到的 javascript 文件
* dynamic linker：浏览器根据 JavaScript 头部声明的 `import {yyy} from './xxx.js'`，动态加载相对当前 url 的 xxx.js

和传统浏览器不同，import 引用的 js 会被浏览器加载，无需用 script 标签 src 引用进来。

```js
// http://localhost/library.js
console.log('i am the library')
```

```html
// http://localhost/index.html
<html>
<head>
<script type="module">
import './library.js'
</script>
</head>
<body>
</body>
</html>
```

用浏览器访问 `http://localhost/index.html` 在浏览器的控制台输出

```
i am the library
```

## nodejs

* executable：直接用参数传递给 node 命令的 js 文件
* dynamic library：js文件 或者 package.json 定义的 js package
* dynamic linker：用 node 提供的全局 require api 加载其他的 js 文件

用 js 文件做为动态链接库

```js
// /opt/executable.js
console.log('i am the executable') 
require('./library.js')
```

```js
// /opt/library.js
console.log('i am the library')
```

```
node /opt/executable.js
// Output:
// i am the executable
// i am the library
```

用包含 package.json 的目录做为动态链接库

目录结构如下

* executable.js
* node_modules
  * library
    * package.json
    * library.js

```js
// /opt/executable.js
console.log('i am the executable') 
require('library')
```

```json
// /opt/node_modules/library/package.json
{ 
  "main": "library.js"
}
```

```js
// /opt/node_modules/library/library.js
console.log('i am the library')
```

```
node /opt/executable.js
// Output:
// i am the executable
// i am the library
```

## 支持 ES6 Module 的 nodejs

* executable：直接用参数传递给 node 命令的 js 文件
* dynamic library：mjs文件 或者 package.json 定义的 js package
* dynamic linker：用 ES6 Module 的 import 语法调用 nodejs 的 require 机制

```js
// /opt/executable.mjs
import './library.mjs' 
console.log('i am the executable')
```

```js
// /opt/library.mjs
console.log('i am the library')
```

```
node --experimental-modules executable.mjs
// Output:
// i am the executable
// i am the library
```

nodejs 使用 ES6 Module 的语法进行动态链接需要两个条件

* 文件名后缀从 .js 变成 .mjs
* 添加 --experimental-modules 的命令行参数

如果 import 的包不是相对路径，也是从 node_modules 里查找 package.json