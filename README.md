# Seneca ：NodeJS 微服务框架入门指南

[Seneca](http://senecajs.org/) 是一个能让您快速构建基于消息的微服务系统的工具集，你不需要知道各种服务本身被部署在何处，不需要知道具体有多少服务存在，也不需要知道他们具体做什么，任何你业务逻辑之外的服务（如数据库、缓存或者第三方集成等）都被隐藏在微服务之后。

这种解耦使您的系统易于连续构建与更新，Seneca 能做到这些，原因在于它的三大核心功能：

1. **模式匹配**：不同于脆弱的服务发现，模式匹配旨在告诉这个世界你真正关心的消息是什么；
2. **无依赖传输**：你可以以多种方式在服务之间发送消息，所有这些都隐藏至你的业务逻辑之后；
3. **组件化**：功能被表示为一组可以一起组成微服务的插件。

在 Seneca 中，消息就是一个可以有任何你喜欢的内部结构的 `JSON` 对象，它们可以通过 HTTP/HTTPS、TCP、消息队列、发布/订阅服务或者任何能传输数据的方式进行传输，而对于作为消息生产者的你来讲，你只需要将消息发送出去即可，完全不需要关心哪些服务来接收它们。

然后，你又想告诉这个世界，你想要接收一些消息，这也很简单，你只需在 Seneca 中作一点匹配模式配置即可，匹配模式也很简单，只是一个键值对的列表，这些键值对被用于匹配 `JSON` 消息的极组属性。

在本文接下来的内容中，我们将一同基于 Seneca 构建一些微服务。

# 模式（ _Patterns_ ）

让我们从一点特别简单的代码开始，我们将创建两个微服务，一个会进行数学计算，另一个去调用它：

```javascript
const seneca = require('seneca')();

seneca.add('role:math, cmd:sum', (msg, reply) => {
  reply(null, { answer: ( msg.left + msg.right )})
});

seneca.act({
  role: 'math',
  cmd: 'sum',
  left: 1,
  right: 2
}, (err, result) => {
  if (err) {
    return console.error(err);
  }
  console.log(result);
});
```

将上面的代码，保存至一个 `js` 文件中，然后执行它，你可能会在 `console` 中看到类似下面这样的消息：

```bash
{"kind":"notice","notice":"hello seneca 4y8daxnikuxp/1483577040151/58922/3.2.2/-","level":"info","when":1483577040175}
(node:58922) DeprecationWarning: 'root' is deprecated, use 'global'
{ answer: 3 }
```

到目前为止，所有这一切都发生在同一个进程中，没有网络流量产生，进程内的函数调用也是基于消息传输。

`seneca.add` 方法，添加了一个新的动作模式（_Action Pattern_）至 `Seneca` 实例中，它有两个参数：

1. `pattern` ：用于匹配 Seneca 实例中 `JSON` 消息体的模式；
2. `action` ：当模式被匹配时执行的操作

`seneca.act` 方法同样有两个参数：

1. `msg` ：作为纯对象提供的待匹配的入站消息；
2. `respond` ：用于接收并处理响应信息的回调函数。

让我们再把所有代码重新过一次：

```javascript
seneca.add('role:math, cmd:sum', (msg, reply) => {
  reply(null, { answer: ( msg.left + msg.right )})
});
```

在上面的代码中的 `Action` 函数，计算了匹配到的消息体中两个属性 `left` 与 `right` 的值的和，并不是所有的消息都会被创建一个响应，但是在绝大多数情况下，是需要有响应的， Seneca 提供了用于响应消息的回调函数。

在匹配模式中， `role:math, cmd:sum` 匹配到了下面这个消息体：

```javascript
{
  role: 'math',
  cmd: 'sum',
  left: 1,
  right: 2
}
```

并得到计自结果：

```javascript
{
  answer: 3
}
```

关于 `role` 与 `cmd` 这两个属性，它们没有什么特别的，只是恰好被你用于匹配模式而已。

接着，`seneca.act` 方法，发送了一条消息，它有两个参数：

1. `msg` ：发送的消息主体
2. `response_callback` ：如果该消息有任何响应，该回调函数都会被执行。

响应的回调函数可接收两个参数： `error` 与 `result` ，如果有任何错误发生（比如，发送出去的消息未被任何模式匹配），则第一个参数将是一个 `Error` 对象，而如果程序按照我们所预期的方向执行了的话，那么，第二个参数将接收到响应结果，在我们的示例中，我们只是简单的将接收到的响应结果打印至了 `console` 而已。

```javascript
seneca.act({
  role: 'math',
  cmd: 'sum',
  left: 1,
  right: 2
}, (err, result) => {
  if (err) {
    return console.error(err);
  }
  console.log(result);
});
```

[sum.js](https://github.com/pantao/getting-started-seneca/blob/master/sum.js) 示例文件，向你展示了如何定义并创建一个 Action 以及如何呼起一个 Action，但它们都发生在一个进程中，接下来，我们很快就会展示如何拆分成不同的代码和多个进程。






# Web 服务集成

Seneca不是一个Web框架。 但是，您仍然需要将其连接到您的Web服务API，你永远要记住的是，不要将你的内部行为模式暴露在外面，这不是一个好的安全的实践，相反的，你应该定义一组API模式，比如用属性 `role：api`，然后你可以将它们连接到你的内部微服务。

下面是我们定义 [api.js](https://github.com/pantao/getting-started-seneca/blob/master/api.js) 插件。

```javascript
module.exports = function api(options) {

  var validOps = { sum:'sum', product:'product' }

  this.add('role:api,path:calculate', function (msg, respond) {
    var operation = msg.args.params.operation
    var left = msg.args.query.left
    var right = msg.args.query.right
    this.act('role:math', {
      cmd:   validOps[operation],
      left:  left,
      right: right,
    }, respond)
  })

  this.add('init:api', function (msg, respond) {
    this.act('role:web',{routes:{
      prefix: '/api',
      pin: 'role:api,path:*',
      map: {
        calculate: { GET:true, suffix:'/{operation}' }
      }
    }}, respond)
  })

}
```

然后，我们使用 `hapi` 作为Web框架，建了 [hapi-app.js](https://github.com/pantao/getting-started-seneca/blob/master/hapi-app.js) 应用：

```javascript
const Hapi = require('hapi');
const Seneca = require('seneca');
const SenecaWeb = require('seneca-web');

const config = {
  adapter: require('seneca-web-adapter-hapi'),
  context: (() => {
    const server = new Hapi.Server();
    server.connection({
      port: 3000
    });

    server.route({
      path: '/routes',
      method: 'get',
      handler: (request, reply) => {
        const routes = server.table()[0].table.map(route => {
          return {
            path: route.path,
            method: route.method.toUpperCase(),
            description: route.settings.description,
            tags: route.settings.tags,
            vhost: route.settings.vhost,
            cors: route.settings.cors,
            jsonp: route.settings.jsonp,
          }
        })
        reply(routes)
      }
    });

    return server;
  })()
};

const seneca = Seneca()
  .use(SenecaWeb, config)
  .use('math')
  .use('api')
  .ready(() => {
    const server = seneca.export('web/context')();
    server.start(() => {
      server.log('server started on: ' + server.info.uri);
    });
  });
```

启动 `hapi-app.js` 之后，访问 <http://localhost:3000/routes>，你便可以看到下面这样的信息：

```javascript
[
  {
    "path": "/routes",
    "method": "GET",
    "cors": false
  },
  {
    "path": "/api/calculate/{operation}",
    "method": "GET",
    "cors": false
  }
]
```

这表示，我们已经成功的将模式匹配更新至 `hapi` 应用的路由中。访问 <http://localhost:3000/api/calculate/sum?left=1&right=2> ，将得到结果：

```javascript
{"answer":3}
```

在上面的示例中，我们直接将 `math` 插件也加载到了 `seneca` 实例中，其实我们可以更加合理的进行这种操作，如 [hapi-app-client.js](https://github.com/pantao/getting-started-seneca/blob/master/hapi-app-client.js) 文件所示：

```javascript
...
const seneca = Seneca()
  .use(SenecaWeb, config)
  .use('api')
  .client({type: 'tcp', pin: 'role:math'})
  .ready(() => {
    const server = seneca.export('web/context')();
    server.start(() => {
      server.log('server started on: ' + server.info.uri);
    });
  });
```

我们不注册 `math` 插件，而是使用 `client` 方法，将 `role:math` 发送给 `math-pin-service.js` 的服务，并且使用的是 `tcp` 连接，没错，你的微服务就是这样成型了。

**注意：永远不要使用外部输入创建操作的消息体，永远显示地在内部创建，这可以有效避免注入攻击。**

在上面的的初始化函数中，调用了一个 `role:web` 的模式操作，并且定义了一个 `routes` 属性，这将定义一个URL地址与操作模式的匹配规则，它有下面这些参数：

- `prefix`：URL 前缀
- `pin`： 需要映射的模式集
- `map`：要用作 URL Endpoint 的 `pin` 通配符属性列表

你的URL地址将开始于 `/api/`。

`rol:api, path:*` 这个 `pin` 表示，映射任何有 `role="api"` 键值对，同时 `path` 属性被定义了的模式，在本例中，只有 `role:api,path:calculate` 符合该模式。

`map` 属性是一个对象，它有一个 `calculate` 属性，对应的URL地址开始于：`/api/calculate`。

按着， `calculate` 的值是一个对象，它表示了 `HTTP` 的 `GET` 方法是被允许的，并且URL应该有参数化的后缀（后缀就类于 `hapi` 的 `route` 规则中一样）。

所以，你的完整地址是 `/api/calculate/{operation}`。

然后，其它的消息属性都将从 URL query 对象或者 JSON body 中获得，在本示例中，因为使用的是 GET 方法，所以没有 body。

`SenecaWeb` 将会通过 `msg.args` 来描述一次请求，它包括：

- `body`：HTTP 请求的 `payload` 部分；
- `query`：请求的 `querystring`；
- `params`：请求的路径参数。

现在，启动前面我们创建的微服务：

```bash
node math-pin-service.js --seneca.log=plugin:math
```

然后再启动我们的应用：

```bash
node hapi-app.js --seneca.log=plugin:web,plugin:api
```

访问下面的地址：

- <http://localhost:3000/api/calculate/product?left=2&right=3> 得到 `{"answer":6}`
- <http://localhost:3000/api/calculate/sum?left=2&right=3> 得到 `{"answer":5}`

# 数据持久化

一个真实的系统，肯定需要持久化数据，在Seneca中，你可以执行任何您喜欢的操作，使用任何类型的数据库层，但是，为什么不使用模式匹配和微服务的力量，使你的开发更轻松？

模式匹配还意味着你可以推迟有关微服务数据的争论，比如服务是否应该"拥有"数据，服务是否应该访问共享数据库等，模式匹配意味着你可以在随后的任何时间重新配置你的系统。

[seneca-entity](https://github.com/senecajs/seneca-entity) 提供了一个简单的数据抽象层（ORM），基于以下操作：

- `load`：根据实体标识加载一个实体；
- `save`：创建或更新（如果你提供了一个标识的话）一个实体；
- `list`：列出匹配查询条件的所有实体；
- `remove`：删除一个标识指定的实体。

它们的匹配模式分别是：

- `load`： `role:entity,cmd:load,name:<entity-name>`
- `save`： `role:entity,cmd:save,name:<entity-name>`
- `list`： `role:entity,cmd:list,name:<entity-name>`
- `remove`： `role:entity,cmd:remove,name:<entity-name>`

任何实现了这些模式的插件都可以被用于提供数据库（比如 [MySQL](https://www.npmjs.com/package/seneca-mysql-store)）访问。

当数据的持久化与其它的一切都基于相同的机制提供时，微服务的开发将变得更容易，而这种机制，便是模式匹配消息。

由于直接使用数据持久性模式可能变得乏味，所以 `seneca` 实体还提供了一个更熟悉的 `ActiveRecord` 风格的接口，要创建记录对象，请调用 `seneca.make` 方法。 记录对象有方法 `load$`、`save$`、`list$` 以及 `remove$`（所有方法都带有 `$` 后缀，以防止与数据字段冲突），数据字段只是对象属性。

通过 `npm` 安装 `seneca-entity`， 然后在你的应用中使用 `seneca.use()` 方法加载至你的 `seneca` 实例。

现在让我们先创建一个简单的数据实体，它保存 `book` 的详情。

文件 [book.js](https://github.com/pantao/getting-started-seneca/blob/master/book.js)

```javascript
const seneca = require('seneca')();
seneca.use('basic').use('entity');

const book = seneca.make('book');
book.title = 'Action in Seneca';
book.price = 9.99;

// 发送 role:entity,cmd:save,name:book 消息
book.save$( console.log );
```

在上面的示例中，我们还使用了 [seneca-basic](https://github.com/senecajs/seneca-basic)，它是 `seneca-entity` 依赖的插件。

执行上面的代码之后，我们可以看到下面这样的日志：

```bash
❯ node book.js
null $-/-/book;id=byo81d;{title:Action in Seneca,price:9.99}
```

> Seneca 内置了 [mem-store](https://www.npmjs.com/package/seneca-mem-store)，这使得我们在本示例中，不需要使用任何其它数据库的支持也能进行完整的数据库持久操作（虽然，它并不是真正的持久化了）。

由于数据的持久化永远都是使用的同样的消息模式集，所以，你可以非常简单的交互数据库，比如，你可能在开发的过程中使用的是 [MongoDB](https://www.npmjs.com/package/seneca-mongo-store)，而后，开发完成之后，在生产环境中使用 [Postgres](https://www.npmjs.com/package/seneca-postgres-store)。

下面让我他创建一个简单的线上书店，我们可以通过它，快速的添加新书、获取书的详细信息以及购买一本书：

[book-store.js](https://github.com/pantao/getting-started-seneca/blob/master/book-store.js)

```javascript
module.exports = function(options) {

  // 从数据库中，查询一本ID为 `msg.id` 的书，我们使用了 `load$` 方法
  this.add('role:store, get:book', function(msg, respond) {
    this.make('book').load$(msg.id, respond);
  });

  // 向数据库中添加一本书，书的数据为 `msg.data`，我们使用了 `data$` 方法
  this.add('role:store, add:book', function(msg, respond) {
    this.make('book').data$(msg.data).save$(respond);
  });

  // 创建一条新的支付订单（在真实的系统中，经常是由商品详情布中的 *购买* 按钮触
  // 发的事件），先是查询出ID为 `msg.id` 的书本，若查询出错，则直接返回错误，
  // 否则，将书本的信息复制给 `purchase` 实体，并保存该订单，然后，我们发送了
  // 一条 `role:store,info:purchase` 消息（但是，我们并不接收任何响应），
  // 这条消息只是通知整个系统，我们现在有一条新的订单产生了，但是我并不关心谁会
  // 需要它。
  this.add('role:store, cmd:purchase', function(msg, respond) {
    this.make('book').load$(msg.id, function(err, book) {
      if (err) return respond(err);

      this
        .make('purchase')
        .data$({
          when: Date.now(),
          bookId: book.id,
          title: book.title,
          price: book.price,
        })
        .save$(function(err, purchase) {
          if (err) return respond(err);

          this.act('role:store,info:purchase', {
            purchase: purchase
          });
          respond(null, purchase);
        });
    });
  });

  // 最后，我们实现了 `role:store, info:purchase` 模式，就只是简单的将信息
  // 打印出来， `seneca.log` 对象提供了 `debug`、`info`、`warn`、`error`、
  // `fatal` 方法用于打印相应级别的日志。
  this.add('role:store, info:purchase', function(msg, respond) {
    this.log.info('purchase', msg.purchase);
    respond();
  });
};
```

接下来，我们可以创建一个简单的单元测试，以验证我们前面创建的程序：

[boot-store-test.js](https://github.com/pantao/getting-started-seneca/blob/master/book-store-test.js)

```javascript
// 使用 Node 内置的 `assert` 模块
const assert = require('assert')

const seneca = require('seneca')()
  .use('basic')
  .use('entity')
  .use('book-store')
  .error(assert.fail)

// 添加一本书
addBook()

function addBook() {
  seneca.act(
    'role:store,add:book,data:{title:Action in Seneca,price:9.99}',
    function(err, savedBook) {

      this.act(
        'role:store,get:book', {
          id: savedBook.id
        },
        function(err, loadedBook) {

          assert.equal(loadedBook.title, savedBook.title)

          purchase(loadedBook);
        }
      )
    }
  )
}

function purchase(book) {
  seneca.act(
    'role:store,cmd:purchase', {
      id: book.id
    },
    function(err, purchase) {
      assert.equal(purchase.bookId, book.id)
    }
  )
}
```

执行该测试：

```bash
❯ node book-store-test.js
["purchase",{"entity$":"-/-/purchase","when":1483607360925,"bookId":"a2mlev","title":"Action in Seneca","price":9.99,"id":"i28xoc"}]
```

在一个生产应用中，我们对于上面的订单数据，可能会有单独的服务进行监控，而不是像上面这样，只是打印一条日志出来，那么，我们现在来创建一个新的服务，用于收集订单数据：

[book-store-stats.js](https://github.com/pantao/getting-started-seneca/blob/master/book-store-stats.js)

```javascript
const stats = {};

require('seneca')()
  .add('role:store,info:purchase', function(msg, respond) {
    const id = msg.purchase.bookId;
    stats[id] = stats[id] || 0;
    stats[id]++;
    console.log(stats);
    respond();
  })
  .listen({
    port: 9003,
    host: 'localhost',
    pin: 'role:store,info:purchase'
  });
```

然后，更新 `book-store-test.js` 文件：

```javascript
const seneca = require('seneca')()
  .use('basic')
  .use('entity')
  .use('book-store')
  .client({port:9003,host: 'localhost', pin:'role:store,info:purchase'})
  .error(assert.fail);
```

此时，当有新的订单产生时，就会通知到订单监控服务了。

## 将所有服务集成到一起

通过上面的所有步骤，我们现在已经有四个服务了：

- [book-store-stats.js](https://github.com/pantao/getting-started-seneca/blob/master/book-store-stats.js) ： 用于收集书店的订单信息；
- [book-store-service.js](https://github.com/pantao/getting-started-seneca/blob/master/book-store-service.js) ：提供书店相关的功能；
- [math-pin-service.js](https://github.com/pantao/getting-started-seneca/blob/master/math-pin-service.js)：提供一些数学相关的服务；
- [app-all.js](https://github.com/pantao/getting-started-seneca/blob/master/app-all.js)：Web 服务

`book-store-stats` 与 `math-pin-service` 我们已经有了，所以，直接启动即可：

```bash
node math-pin-service.js --seneca.log.all
node book-store-stats.js --seneca.log.all
```

现在，我们需要一个 `book-store-service` ：

```javascript
require('seneca')()
  .use('basic')
  .use('entity')
  .use('book-store')
  .listen({
    port: 9002,
    host: 'localhost',
    pin: 'role:store'
  })
  .client({
    port: 9003,
    host: 'localhost',
    pin: 'role:store,info:purchase'
  });
```

该服务接收任何 `role:store` 消息，但同时又将任何 `role:store,info:purchase` 消息发送至网络，**永远都要记住， client 与 listen 的 pin 配置必须完全一致**。

现在，我们可以启动该服务：

```bash
node book-store-service.js --seneca.log.all
```

然后，创建我们的 `app-all.js`，首选，复制 `api.js` 文件到 [api-all.js](https://github.com/pantao/getting-started-seneca/blob/master/api-all.js)，这是我们的API。

```javascript
module.exports = function api(options) {

  var validOps = {
    sum: 'sum',
    product: 'product'
  }

  this.add('role:api,path:calculate', function(msg, respond) {
    var operation = msg.args.params.operation
    var left = msg.args.query.left
    var right = msg.args.query.right
    this.act('role:math', {
      cmd: validOps[operation],
      left: left,
      right: right,
    }, respond)
  });

  this.add('role:api,path:store', function(msg, respond) {
    let id = null;
    if (msg.args.query.id) id = msg.args.query.id;
    if (msg.args.body.id) id = msg.args.body.id;

    const operation = msg.args.params.operation;
    const storeMsg = {
      role: 'store',
      id: id
    };
    if ('get' === operation) storeMsg.get = 'book';
    if ('purchase' === operation) storeMsg.cmd = 'purchase';
    this.act(storeMsg, respond);
  });

  this.add('init:api', function(msg, respond) {
    this.act('role:web', {
      routes: {
        prefix: '/api',
        pin: 'role:api,path:*',
        map: {
          calculate: {
            GET: true,
            suffix: '/{operation}'
          },
          store: {
            GET: true,
            POST: true,
            suffix: '/{operation}'
          }
        }
      }
    }, respond)
  })

}
```

最后， [app-all.js](https://github.com/pantao/getting-started-seneca/blob/master/app-all.js)：

```javascript
const Hapi = require('hapi');
const Seneca = require('seneca');
const SenecaWeb = require('seneca-web');

const config = {
  adapter: require('seneca-web-adapter-hapi'),
  context: (() => {
    const server = new Hapi.Server();
    server.connection({
      port: 3000
    });

    server.route({
      path: '/routes',
      method: 'get',
      handler: (request, reply) => {
        const routes = server.table()[0].table.map(route => {
          return {
            path: route.path,
            method: route.method.toUpperCase(),
            description: route.settings.description,
            tags: route.settings.tags,
            vhost: route.settings.vhost,
            cors: route.settings.cors,
            jsonp: route.settings.jsonp,
          }
        })
        reply(routes)
      }
    });

    return server;
  })()
};

const seneca = Seneca()
  .use(SenecaWeb, config)
  .use('basic')
  .use('entity')
  .use('math')
  .use('api-all')
  .client({
    type: 'tcp',
    pin: 'role:math'
  })
  .client({
    port: 9002,
    host: 'localhost',
    pin: 'role:store'
  })
  .ready(() => {
    const server = seneca.export('web/context')();
    server.start(() => {
      server.log('server started on: ' + server.info.uri);
    });
  });

// 创建一本示例书籍
seneca.act(
  'role:store,add:book', {
    data: {
      title: 'Action in Seneca',
      price: 9.99
    }
  },
  console.log
)
```

启动该服务：

```bash
node app-all.js --seneca.log.all
```

从控制台我们可以看到下面这样的消息：

```bash
null $-/-/book;id=0r7mg7;{title:Action in Seneca,price:9.99}
```

这表示成功创建了一本ID为 `0r7mg7` 的书籍，现在，我们访问 [http://localhost:3000/api/store/get?id=0r7mg7](http://localhost:3000/api/store/get?id=0r7mg7) 即可查看该ID的书籍详情（ID是随机的，所以，你生成的ID可能并不是这样的）。

[http://localhost:3000/routes](http://localhost:3000/routes) 可以查看所有的路由。

然后我们可创建一个新的购买订单：

```bash
curl -d '{"id":"0r7mg7"}' -H "content-type:application/json" http://localhost:3000/api/store/purchase
{"when":1483609872715,"bookId":"0r7mg7","title":"Action in Seneca","price":9.99,"id":"8suhf4"}
```

访问 [http://localhost:3000/api/calculate/sum?left=2&right=3](http://localhost:3000/api/calculate/sum?left=2&right=3) 可以得到 `{"answer":5}`。

## 最佳 Seneca 应用结构实践

### 推荐你这样做

-   将业务逻辑与执行分开，放在单独的插件中，比如不同的Node模块、不同的项目甚至同一个项目下不同的文件都是可以的；

-   使用执行脚本撰写您的应用程序，不要害怕为不同的上下文使用不同的脚本，它们看上去应该很短，比如像下面这样：

    ```javascript
    var SOME_CONFIG = process.env.SOME_CONFIG || 'some-default-value'

    require('seneca')({ some_options: 123 })

      // 已存在的 Seneca 插件
      .use('community-plugin-0')
      .use('community-plugin-1', {some_config: SOME_CONFIG})
      .use('community-plugin-2')

      // 业务逻辑插件
      .use('project-plugin-module')
      .use('../plugin-repository')
      .use('./lib/local-plugin')

      .listen( ... )
      .client( ... )

      .ready( function() {
        // 当 Seneca 启动成功之后的自定义脚本
      })
    ```

-   插件加载顺序很重要，这当然是一件好事，可以主上你对消息的成有绝对的控制权。

### 不推荐你这样做

-   将 Seneca 应用的启动与初始化同其它框架的启动与初始化放在一起了，永远记住，保持事务的简单；
-   将 Seneca 实例当做变量到处传递。
