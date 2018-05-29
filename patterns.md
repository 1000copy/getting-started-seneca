# 匹配模式如何工作？

模式----而不是网络地址或者会话，让你可以更加容易的扩展或增强您的系统，这样做，让添加新的微服务变得更简单。

现在让我们给系统再添加一个新的功能----计算两个数字的乘积。

我们想要发送的消息看起来像下面这样的：

```javascript
{
  role: 'math',
  cmd: 'product',
  left: 3,
  right: 4
}
```

而后获得的结果看起来像下面这样的：

```javascript
{
  answer: 12
}
```

知道怎么做了吧？你可以像 `role: math, cmd: sum` 模式这样，创建一个 `role: math, cmd: product` 操作：

```javascript
seneca.add('role:math, cmd:product', (msg, reply) => {
  reply(null, { answer: ( msg.left * msg.right )})
});
```

然后，调用该操作：

```javascript
seneca.act({
  role: 'math',
  cmd: 'product',
  left: 3,
  right: 4
}, (err, result) => {
  if (err) {
    return console.error(err);
  }
  console.log(result);
});
```

运行 [product.js](https://github.com/pantao/getting-started-seneca/blob/master/product.js) ，你将得到你想要的结果。

将这两个方法放在一起，代码像是下面这样的：

```javascript
const seneca = require('seneca')();

seneca.add('role:math, cmd:sum', (msg, reply) => {
  reply(null, { answer: ( msg.left + msg.right )})
});

seneca.add('role:math, cmd:product', (msg, reply) => {
  reply(null, { answer: ( msg.left * msg.right )})
});

seneca.act({role: 'math', cmd: 'sum', left: 1, right: 2}, console.log)
      .act({role: 'math', cmd: 'product', left: 3, right: 4}, console.log)
```

运行 [sum-product.js](https://github.com/pantao/getting-started-seneca/blob/master/sum-product.js) 后，你将得到下面这样的结果：

```bash
null { answer: 3 }
null { answer: 12 }
```

在上面合并到一起的代码中，我们发现， `seneca.act` 是可以进行链式调用的，`Seneca` 提供了一个链式API，调式调用是顺序执行的，但是不是串行，所以，返回的结果的顺序可能与调用顺序并不一样。

# 扩展模式以增加新功能

模式让你可以更加容易的扩展程序的功能，与 `if...else...` 语法不同的是，你可以通过增加更多的匹配模式以达到同样的功能。

下面让我们扩展一下 `role: math, cmd: sum` 操作，它只接收整型数字，那么，怎么做？

```javascript
seneca.add({role: 'math', cmd: 'sum', integer: true}, function (msg, respond) {
  var sum = Math.floor(msg.left) + Math.floor(msg.right)
  respond(null, {answer: sum})
})
```

现在，下面这条消息：

```javascript
{role: 'math', cmd: 'sum', left: 1.5, right: 2.5, integer: true}
```

将得到下面这样的结果：

```javascript
{answer: 3}  // == 1 + 2，小数部分已经被移除了
```

代码可在 [sum-integer.js](https://github.com/pantao/getting-started-seneca/blob/master/sum-integer.js) 中查看。

现在，你的两个模式都存在于系统中了，而且还存在交叉部分，那么 `Seneca` 最终会将消息匹配至哪条模式呢？原则是：更多匹配项目被匹配到的优先，被匹配到的属性越多，则优先级越高。

[pattern-priority-testing.js](https://github.com/pantao/getting-started-seneca/blob/master/pattern-priority-testing.js) 可以给我们更加直观的测试：

```javascript
const seneca = require('seneca')()

seneca.add({role: 'math', cmd: 'sum'}, function (msg, respond) {
  var sum = msg.left + msg.right
  respond(null, {answer: sum})
})

// 下面两条消息都匹配 role: math, cmd: sum

seneca.act({role: 'math', cmd: 'sum', left: 1.5, right: 2.5}, console.log)
seneca.act({role: 'math', cmd: 'sum', left: 1.5, right: 2.5, integer: true}, console.log)

setTimeout(() => {
  seneca.add({role: 'math', cmd: 'sum', integer: true}, function (msg, respond) {
    var sum = Math.floor(msg.left) + Math.floor(msg.right)
    respond(null, { answer: sum })
  })

  // 下面这条消息同样匹配 role: math, cmd: sum
  seneca.act({role: 'math', cmd: 'sum', left: 1.5, right: 2.5}, console.log)

  // 但是，也匹配 role:math,cmd:sum,integer:true
  // 但是因为更多属性被匹配到，所以，它的优先级更高
  seneca.act({role: 'math', cmd: 'sum', left: 1.5, right: 2.5, integer: true}, console.log)
}, 100)
```

输出结果应该像下面这样：

```bash
null { answer: 4 }
null { answer: 4 }
null { answer: 4 }
null { answer: 3 }
```

在上面的代码中，因为系统中只存在 `role: math, cmd: sum` 模式，所以，都匹配到它，但是当 100ms 后，我们给系统中添加了一个 `role: math, cmd: sum, integer: true` 模式之后，结果就不一样了，匹配到更多的操作将有更高的优先级。

这种设计，可以让我们的系统可以更加简单的添加新的功能，不管是在开发环境还是在生产环境中，你都可以在不需要修改现有代码的前提下即可更新新的服务，你只需要先好新的服务，然后启动新服务即可。

# 基于模式的代码复用

模式操作还可以调用其它的操作，所以，这样我们可以达到代码复用的需求：

```javascript
const seneca = require('seneca')()

seneca.add('role: math, cmd: sum', function (msg, respond) {
  var sum = msg.left + msg.right
  respond(null, {answer: sum})
})

seneca.add('role: math, cmd: sum, integer: true', function (msg, respond) {
  // 复用 role:math, cmd:sum
  this.act({
    role: 'math',
    cmd: 'sum',
    left: Math.floor(msg.left),
    right: Math.floor(msg.right)
  }, respond)
})

// 匹配 role:math,cmd:sum
seneca.act('role: math, cmd: sum, left: 1.5, right: 2.5',console.log)

// 匹配 role:math,cmd:sum,integer:true
seneca.act('role: math, cmd: sum, left: 1.5, right: 2.5, integer: true', console.log)
```

在上面的示例代码中，我们使用了 `this.act` 而不是前面的 `seneca.act`，那是因为，在 `action` 函数中，上下文关系变量 `this` ，引用了当前的 `seneca` 实例，这样你就可以在任何一个 `action` 函数中，访问到该 `action` 调用的整个上下文。

在上面的代码中，我们使用了 JSON 缩写形式来描述模式与消息， 比如，下面是对象字面量：

```javascript
{role: 'math', cmd: 'sum', left: 1.5, right: 2.5}
```

缩写模式为：

```text
'role: math, cmd: sum, left: 1.5, right: 2.5'
```

[jsonic](https://github.com/rjrodger/jsonic) 这种格式，提供了一种以字符串字面量来表达对象的简便方式，这使得我们可以创建更加简单的模式和消息。

上面的代码保存在了 [sum-reuse.js](https://github.com/pantao/getting-started-seneca/blob/master/sum-reuse.js) 文件中。

# 模式是唯一的

你定义的 Action 模式都是唯一了，它们只能触发一个函数，模式的解析规则如下：

- 更多我属性优先级更高
- 若模式具有相同的数量的属性，则按字母顺序匹配

规则被设计得很简单，这使得你可以更加简单的了解到到底是哪个模式被匹配了。

下面这些示例可以让你更容易理解：

- `a: 1, b: 2` 优先于 `a: 1`， 因为它有更多的属性；
- `a: 1, b: 2` 优先于 `a: 1, c: 3`，因为 `b` 在 `c` 字母的前面；
- `a: 1, b: 2, d: 4` 优先于 `a: 1, c: 3, d:4`，因为 `b` 在 `c` 字母的前面；
- `a: 1, b:2, c:3` 优先于 `a:1, b: 2`，因为它有更多的属性；
- `a: 1, b:2, c:3` 优先于 `a:1, c:3`，因为它有更多的属性。

很多时间，提供一种可以让你不需要全盘修改现有 Action 函数的代码即可增加它功能的方法是很有必要的，比如，你可能想为某一个消息增加更多自定义的属性验证方法，捕获消息统计信息，添加额外的数据库结果中，或者控制消息流速等。

我下面的示例代码中，加法操作期望 `left` 和 `right` 属性是有限数，此外，为了调试目的，将原始输入参数附加到输出的结果中也是很有用的，您可以使用以下代码添加验证检查和调试信息：

```javascript
const seneca = require('seneca')()

seneca
  .add(
    'role:math,cmd:sum',
    function(msg, respond) {
      var sum = msg.left + msg.right
      respond(null, {
        answer: sum
      })
    })

// 重写 role:math,cmd:sum with ，添加额外的功能
.add(
  'role:math,cmd:sum',
  function(msg, respond) {

    // bail out early if there's a problem
    if (!Number.isFinite(msg.left) ||
      !Number.isFinite(msg.right)) {
      return respond(new Error("left 与 right 值必须为数字。"))
    }

    // 调用上一个操作函数 role:math,cmd:sum
    this.prior({
      role: 'math',
      cmd: 'sum',
      left: msg.left,
      right: msg.right,

    }, function(err, result) {
      if (err) return respond(err)

      result.info = msg.left + '+' + msg.right
      respond(null, result)
    })
  })

// 增加了的 role:math,cmd:sum
.act('role:math,cmd:sum,left:1.5,right:2.5',
  console.log // 打印 { answer: 4, info: '1.5+2.5' }
)
```

`seneca` 实例提供了一个名为 `prior` 的方法，让可以在当前的 `action` 方法中，调用被其重写的旧操作函数。

`prior` 函数接受两个参数：

1. `msg`：消息体
2. `response_callback`：回调函数

在上面的示例代码中，已经演示了如何修改入参与出参，修改这些参数与值是可选的，比如，可以再添加新的重写，以增加日志记录功能。

在上面的示例中，也同样演示了如何更好的进行错误处理，我们在真正进行操作之前，就验证的数据的正确性，若传入的参数本身就有错误，那么我们直接就返回错误信息，而不需要等待真正计算的时候由系统去报错了。

> 错误消息应该只被用于描述错误的输入或者内部失败信息等，比如，如果你执行了一些数据库的查询，返回没有任何数据，这并不是一个错误，而仅仅只是数据库的事实的反馈，但是如果连接数据库失败，那就是一个错误了。

上面的代码可以在 [sum-valid.js](https://github.com/pantao/getting-started-seneca/blob/master/sum-valid.js) 文件中找到。
