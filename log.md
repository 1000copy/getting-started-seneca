Seneca 还提供了详细日志记录功能，可以提供为开发或者生产提供更多的日志信息，通常的，日志级别被设置为 `INFO`，它并不会打印太多日志信息，如果想看到所有的日志信息，试试以下面这样的方式启动你的服务：

```bash
node minimal-plugin.js --seneca.log.all
```

会不会被吓一跳？当然，你还可以过滤日志信息：

```bash
node minimal-plugin.js --seneca.log.all | grep plugin:define
```

通过日志我们可以看到， seneca 加载了很多内置的插件，比如 `basic`、`transport`、`web` 以及 `mem-store`，这些插件为我们提供了创建微服务的基础功能，同样，你应该也可以看到 `minimal_plugin` 插件。
