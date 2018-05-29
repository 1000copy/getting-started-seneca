# 插件

Seneca插件就只是一个具有单个参数选项的函数，你将这个插件定义函数传递给 `seneca.use` 方法，下面这个是最小的Seneca插件：

    // filename : minimal-plugin.js
    function minimal_plugin(options) {
      console.log(options)
    }
    require('seneca')()
      .use(minimal_plugin, {foo: 'bar'})

`seneca.use` 方法接受两个参数：

1. `plugin` ：插件定义函数或者一个插件名称；
2. `options` ：插件配置选项

上面的示例代码执行后，打印出来的日志看上去是这样的：

    {"kind":"notice","notice":"hello seneca 3qk0ij5t2bta/1483584697034/62768/3.2.2/-","level":"info","when":1483584697057}
    (node:62768) DeprecationWarning: 'root' is deprecated, use 'global'
    { foo: 'bar' }


现在，让我们为这个插件添加一些操作模式：

    //math-plugin.js
    function math(options) {
      this.add('role:math,cmd:sum', function (msg, respond) {
        respond(null, { answer: msg.left + msg.right })
      })
      this.add('role:math,cmd:product', function (msg, respond) {
        respond(null, { answer: msg.left * msg.right })
      })
    }
    require('seneca')()
      .use(math)
      .act('role:math,cmd:sum,left:1,right:2', console.log)


运行得到下面这样的信息：

    null { answer: 3 }


## 插件初始化

插件通过需要进行一些初始化的工作，比如连接数据库等，要初始化插件，你需要定义一个特殊的匹配模式 `init: <plugin-name>`，对于每一个插件，将按顺序调用此操作模式，`init` 函数必须调用其 `callback` 函数，并且不能有错误发生，如果插件初始化失败，则 Seneca 会立即退出 Node 进程。所以的插件初始化工作都必须在任何操作执行之前完成。

为了演示初始化，让我们向 `math` 插件添加简单的自定义日志记录，当插件启动时，它打开一个日志文件，并将所有操作的日志写入文件，文件需要成功打开并且可写，如果这失败，微服务启动就应该失败。


    const fs = require('fs')
    function math(options) {
      var log
      this.add('role:math,cmd:sum',     sum)
      this.add('role:math,cmd:product', product)
      this.add('init:math', init)
      function init(msg, respond) {
        fs.open(options.logfile, 'a', function (err, fd) {
          if (err) return respond(err)
          log = makeLog(fd)
          respond()
        })
      }
      function sum(msg, respond) {
        var out = { answer: msg.left + msg.right }
        log('sum '+msg.left+'+'+msg.right+'='+out.answer+'\n')
        respond(null, out)
      }
      function product(msg, respond) {
        var out = { answer: msg.left * msg.right }
        log('product '+msg.left+'*'+msg.right+'='+out.answer+'\n')
        respond(null, out)
      }
      function makeLog(fd) {
        return function (entry) {
          fs.write(fd, new Date().toISOString()+' '+entry, null, 'utf8', function (err) {
            if (err) return console.log(err)
            fs.fsync(fd, function (err) {
              if (err) return console.log(err)
            })
          })
        }
      }
    }
    require('seneca')()
      .use(math, {logfile:'./math.log'})
      .act('role:math,cmd:sum,left:1,right:2', console.log)

要查看失败时的操作，可以尝试将日志文件位置更改为无效的，例如 `/math.log`。