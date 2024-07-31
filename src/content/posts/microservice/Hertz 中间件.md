---
title: Hertz 中间件详解
published: 2024-07-31 15:00:00
description: 'The first step of system design.'
image: ''
tags: ['Microservice', 'Golang', 'Hertz']
category: 'Microservice'
draft: false 

---

>   Hertz 的中间件设计与 Gin 几乎一样，如果有过编写 Gin middleware 的经历，那么可以直接跳过不看。

中间件是微服务中的一个基本概念，本质上是为了解耦单体架构项目中的不同服务而引入的外部组件。但我们这里提到的中间件，其实是 Web 服务框架中请求调用链中的具备特殊功能（比如认证、鉴权）的方法（函数）。

中间件按照调用链发起方可以划分为两类：

-   服务端中间件；
-   客户端中间件。

## 服务端中间件

Hertz 服务端的中间件根据是否与业务逻辑的 `real handle` 处于同一函数调用栈划分为两类：

1.   **pre-handle**：与 real handle 处于不同函数调用栈的中间件；
2.   **post-handle**：与 real handle 处于相同函数调用栈的中间件。

![image-20240731135717896](https://raw.githubusercontent.com/MasakiMu319/fuwari/main/src/assets/post-images/202407311426857.png)

两者的区别从代码实现上为是否需要显示调用 `c.Next(ctx)`

```go
// 方式一
func MyMiddleware() app.HandlerFunc {
  return func(ctx context.Context, c *app.RequestContext) {
    // pre-handle
    // ...
    c.Next(ctx)
  }
}

// 方式二
func MyMiddleware() app.HandlerFunc {
  return func(ctx context.Context, c *app.RequestContext) {
    c.Next(ctx) // call the next middleware(handler)
    // post-handle
    // ...
  }
}
```

-   ctx：是 Go 中原生的上下文管理对象，主要用于传递跨请求范围的数据；
-   c：是 Hertz 框架中传递上下文的对象，专注请求的处理逻辑和响应的生成，例如上面的代码中从当前中间件使用 `c.Next()` 方法显式调用下一个中间件。

:::note

严格来说，是否使用 c.Next(ctx) 并不是区分的关键，或者没有区分的必要。中间件的使用相当于就是在原有的请求调用链路上新增一个节点，如果这个节点没有显式使用 c.Abort() 等方法退出函数调用链，那么，`return` 和 `c.Next` 都会默认执行当前调用链后续的中间件。但是 c.Next 的不同之处在于将控制权让渡给后续调用链上的中间件，后续还会回到 `c.Next` 后继续执行，而 `return` 的行为是直接结束当前中间件的执行，就是函数执行完了。

👇这段代码来自 Hertz：c.Next 部分调用的源码：

```go
// Next should be used only inside middleware.
// It executes the pending handlers in the chain inside the calling handler.
func (ctx *RequestContext) Next(c context.Context) {
  // 当前中间件执行完成，计数下一个需要执行的中间件位置；
	ctx.index++
  // 将中间件 c.Next 后的逻辑暂停，执行后续的其他中间件；
	for ctx.index < int8(len(ctx.handlers)) {
    // 函数是一等公民，所以直接通过在 handlers 中存放中间件来获取下一个需要调用的中间件；
		ctx.handlers[ctx.index](c, ctx)
		ctx.index++
	}
}
```

:::

## 客户端中间件

后续补充...
