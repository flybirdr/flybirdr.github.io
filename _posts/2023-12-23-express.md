
资料来源：[链接](https://juejin.cn/post/7238844861532504101)
#### 基本使用
```shell
# 创建文件夹
mkdir study-express

# 进入文件夹
cd study-express

# 初始化项目，创建package.json文件。一路回车即可
npm init

# 安装express依赖
npm install express
```

```js
// index.js
const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => {
  res.send('Hello World!')
})

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`)
})

```

```shell
node index.js
```

```js
...
const bodyParser = require('body-parser')
const jsonParser = bodyParser.json()
...

app.post('/age', jsonParser, (req, res) => {
    const {name, birthday} = req.body;
    const age = new Date().getFullYear() - Number(birthday);
    res.send(`Hello ${name}, Your age is ${age}`)
})
```
#### 静态资源
```shell
# 创建一个名为public的文件夹
mkdir public

# 进入到这个文件夹
cd public

# 创建一个名为public的文件夹

mkdir images

mkdir pdfs
```

```js
// 引用资源
// index.js
app.use(express.static('public'));

// 多个资源
// index.js
app.use(express.static('public'));
app.use(express.static('files'));

//虚拟路径
app.use('/static', express.static('public'));
app.use('/static', express.static('files'));
```

#### 中间件

```js
var doSomething = function(req, res, next) {
    // 在这里执行需要的操作
    consonle.log('Do something in there');
    // 将控制权传递给下一个中间件函数
    next();
}
```
在装入中间件函数时，我们一般有以下两种方式：

* `app.use()`

* `app.METHOD()`

```js
// 方式一：
var logChinaDate = function(req, res, next) {
    console.log(`China\'s Date is ${new Date().toLocaleDateString()}`);
    next();
}
app.use("/china/:province", logChinaDate);
```
```js
var addReqTime = function(req, res, next) {
    const now = new Date();
    req.request_time = now;
    next();
}
app.post('/age', addReqTime);
...
// 以下代码是在之前已经编写过的，放在这里仅仅是为了展示如何处理多个中间件函数
app.post('/age', jsonParser, (req, res) => {
    console.log({request_time: req.request_time});
    
    const {name, birthday = 2000} = req.body;
    const age = new Date().getFullYear() - Number(birthday);
    res.send(`Hello ${name}, Your age is ${age}`)
})

// 方式二：
app.post('/age', addReqTime, jsonParser, (req, res) => {
    console.log({request_time: req.request_time});
    
    const {name, birthday = 2000} = req.body;
    const age = new Date().getFullYear() - Number(birthday);
    res.send(`Hello ${name}, Your age is ${age}`)
})

```
在Express官方文档中，把中间件分为了5个类型：

* 应用层中间件
* 路由器层中间件
* 错误处理中间件
* 内置中间件
* 第三方中间件：

路由中间件：
```js
// 创建了一个`express.Router()` 的实例
var router = express.Router();

// 这里只是一个例子，router的用法与前面生成的express实例app的用法完全一致
router.get('/', (req, res) => {
    res.send('Hello World!')
})

// 将实例router挂载到我们的服务器应用上
app.use('/', router);
```
错误处理中间件：
```js
app.use(function(err, req, res, next) {
  console.error(err.stack);
  res.status(500).send('Something broke!');
});
```
内置中间件：[链接](https://github.com/senchalabs/connect#middleware)

第三方中间件：[链接](https://expressjs.com/zh-cn/resources/middleware.html)

错误处理：

在前面的错误处理中间件中，我们处理了服务器异常时的情况（`500错误`），但是错误处理中间件并不能处理`客户端请求一个不存在资源的情况`，也就是我们常说的`404错误`。

在 Express 中，404 响应不是错误的结果，所以错误处理程序中间件不会将其捕获。

404 响应其实是：Express 执行了所有中间件函数和路由，且发现它们都没有响应。

针对于404响应的情况，我们只需要`在堆栈的最底部（在其他所有函数之下）添加一个中间件函数来处理 404 响应`，我们添加一个简单的中间件函数，来处理`客户端请求一个不存在资源的情况`：

```js
...
// 这个中间件需要添加在所有函数之后
app.use(function(req, res, next) {
    res.status(404).send('Sorry cant find that!');
});
```

#### 接入数据库

推荐使用Docker来配置数据库，开箱即用又能避免污染本地环境。

使用`mysql:8.0.35-debian`这个镜像会出现以下问题，`nodejs`连接会报错.

> https://stackoverflow.com/questions/50093144/mysql-8-0-client-does-not-support-authentication-protocol-requested-by-server
>


原因是因为`mysql8`之前版本加密规则是`mysql_native_password`，之后是`caching_sha2_password`

```mysql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'z';

FLUSH PRIVILEGES; 

quit
```

我这里使用命令设置了`mysql_native_password`，依然无效。

使用`mariadb:latest`这个镜像，`nodejs`可以正常连接。

```shell
mariadb -u root -p
```

```mariadb
# 查看数据库
show databases;
# 创建数据库
create database mydb;
# 选择数据库
use mydb;
# 创建用户表
create table user(
user_id INT NOT NULL AUTO_INCREMENT,
user_name VARCHAR(100) NOT NULL,
user_password VARCHAR(100) NOT NULL,
primary key (user_id)
)ENGINE=Innodb DEFAULT CHARSET=utf8;
# 查询
select * from user;
# 插入一条内容
insert into user (user_name, user_password) values ('admin','admin1234');
# 查询
select * from user;
```



```shell
npm install mysql
mkdir db
```

```js
// db/indes.js
const mysql = require('mysql')

// 注意：这里要根据自己数据库中的配置来填写！
const connection = mysql.createConnection({
  host: 'localhost',
  port: 3306,
  user: 'root',
  password: 'z',
  database: 'mydb'
})
connection.connect()

module.exports = connection;
```



```js
// index.js
...
const db = require("./db/index");
...
```

```js
// index.js
app.get('/users', (req, res) => {
    db.query('select * from user', (err, users) => {
        if (err) throw err
        res.json(users);
    })
})
```



