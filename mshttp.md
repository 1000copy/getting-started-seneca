# 创建微服务

现在让我们把 `math` 插件变成一个真正的微服务。

将业务逻辑（即插件定义）放在其自己的文件中是有意义的。 内容如下：
  
    // math.js
    module.exports = function math(options) {
      this.add('role:math,cmd:sum', function sum(msg, respond) {
        respond(null, { answer: msg.left + msg.right })
      })
      this.add('role:math,cmd:product', function product(msg, respond) {
        respond(null, { answer: msg.left * msg.right })
      })
    }

然后，我们可以引用此服务，：

    require('seneca')()
      .use('math') // 在当前目录下找到 `./math.js`
      .act('role:math,cmd:sum,left:1,right:2', console.log)

但是，到现在为止，所有的操作都还存在于同一个进程中。

## http微服务

接下来，让我们先创建文件，填入以下内容：

    // [math-service.js]
    require('seneca')()
      .use('math')
      .listen()

然后启动该脚本，即可启动我们的微服务，它会启动一个进程监听HTTP请求，端口号为 `10101` 。

试试这样命令：

    curl -d '{"role":"math","cmd":"sum","left":1,"right":2}' http://localhost:10101/act

可以看到结果：

    {"answer":3}


## 访问微服务

真实应用中，一般是使用js代码来访问微服务的。代码如下：

    //[math-client.js]
    require('seneca')()
      .client()
      .act('role:math,cmd:sum,left:1,right:2',console.log)

打开一个新的终端，执行该脚本：

    null { answer: 3 } { id: '7uuptvpf8iff/9wfb26kbqx55',
      accept: '043di4pxswq7/1483589685164/65429/3.2.2/-',
      track: undefined,
      time:
       { client_sent: '0',
         listen_recv: '0',
         listen_sent: '0',
         client_recv: 1483589898390 } }


在 `Seneca` 中，我们通过 `seneca.listen` 方法创建微服务，然后通过 `seneca.client` 去与微服务进行通信。在上面的示例中，我们使用的都是 Seneca 的默认配置，比如 `HTTP` 协议监听 `10101` 端口，但 `seneca.listen` 与 `seneca.client` 方法都可以接受下面这些参数，以达到定抽的功能：

- `port` ：可选的数字，表示端口号；
- `host` ：可先的字符串，表示主机名或者IP地址；
- `spec` ：可选的对象，完整的定制对象

> **注意**：在 Windows 系统中，如果未指定 `host`， 默认会连接 `0.0.0.0`，这是没有任何用处的，你可以设置 `host` 为 `localhost`。

只要 `client` 与 `listen` 的端口号与主机一致，它们就可以进行通信：

- seneca.client(8080) → seneca.listen(8080)
- seneca.client(8080, '192.168.0.2') → seneca.listen(8080, '192.168.0.2')
- seneca.client({ port: 8080, host: '192.168.0.2' }) → seneca.listen({ port: 8080, host: '192.168.0.2' })

Seneca 为你提供的 **无依赖传输** 特性，让你在进行业务逻辑开发时，不需要知道消息如何传输或哪些服务会得到它们，而是在服务设置代码或配置中指定，比如 `math.js` 插件中的代码永远不需要改变，我们就可以任意的改变传输方式。

虽然 `HTTP` 协议很方便，但是并不是所有时间都合适，另一个常用的协议是 `TCP`，我们可以很容易的使用 `TCP` 协议来进行数据的传输，尝试下面这两个文件：

[math-service-tcp.js](https://github.com/pantao/getting-started-seneca/blob/master/math-service-tcp.js) :


require('seneca')()
  .use('math')
  .listen({type: 'tcp'})


[math-client-tcp.js](https://github.com/pantao/getting-started-seneca/blob/master/math-client-tcp.js)


require('seneca')()
  .client({type: 'tcp'})
  .act('role:math,cmd:sum,left:1,right:2',console.log)


默认情况下， `client/listen` 并未指定哪些消息将发送至哪里，只是本地定义了模式的话，会发送至本地的模式中，否则会全部发送至服务器中，我们可以通过一些配置来定义哪些消息将发送到哪些服务中，你可以使用一个 `pin` 参数来做这件事情。

让我们来创建一个应用，它将通过 TCP 发送所有 `role:math` 消息至服务，而把其它的所有消息都在发送至本地：

[math-pin-service.js](https://github.com/pantao/getting-started-seneca/blob/master/math-pin-service.js)：


require('seneca')()

  .use('math')

  // 监听 role:math 消息
  // 重要：必须匹配客户端
  .listen({ type: 'tcp', pin: 'role:math' })


[math-pin-client.js](https://github.com/pantao/getting-started-seneca/blob/master/math-pin-client.js)：


require('seneca')()

  // 本地模式
  .add('say:hello', function (msg, respond){ respond(null, {text: "Hi!"}) })

  // 发送 role:math 模式至服务
  // 注意：必须匹配服务端
  .client({ type: 'tcp', pin: 'role:math' })

  // 远程操作
  .act('role:math,cmd:sum,left:1,right:2',console.log)

  // 本地操作
  .act('say:hello',console.log)


你可以通过各种过滤器来自定义日志的打印，以跟踪消息的流动，使用 `--seneca...` 参数，支持以下配置：

- `date-time`： log 条目何时被创建；
- `seneca-id`： Seneca process ID；
- `level`：`DEBUG`、`INFO`、`WARN`、`ERROR` 以及 `FATAL` 中任何一个；
- `type`：条目编码，比如 `act`、`plugin` 等；
- `plugin`：插件名称，不是插件内的操作将表示为 `root$`；
- `case`： 条目的事件：`IN`、`ADD`、`OUT` 等
- `action-id/transaction-id`：跟踪标识符，_在网络中永远保持一致_；
- `pin`：`action` 匹配模式；
- `message`：入/出参消息体

如果你运行上面的进程，使用了 `--seneca.log.all`，则会打印出所有日志，如果你只想看 `math` 插件打印的日志，可以像下面这样启动服务：

bash
node math-pin-service.js --seneca.log=plugin:math
