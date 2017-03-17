# lapis请求处理
每个被`Lapis`处理的`HTTP`请求在被`Nginx`处理后都遵循相同的基本流程。第一步是路由。路由是 `url` 必须匹配的模式。当你定义一个路由时，你也得包括一个处理函数。这个处理函数是一个常规的`Lua/MoonScript`函数，如果相关联的路由匹配，则将调用该函数。

所有被调用的处理函数都具有一个参数（一个请求对象）。请求对象将存储您希望在处理函数和视图之间共享的所有数据。此外，请求对象是您向`Web服务器`了解如何将结果发送到客户端的接口。

处理函数的返回值用于渲染输出。**字符串返回值将直接呈现给浏览器**。`table` 的返回值将用作[渲染选项]()。如果有多个返回值，则所有这些返回值都合并到最终结果中。您可以返回字符串和`table`以控制输出。

如果没有匹配请求的路由，则执行默认路由处理程序，在[application callbacks]()了解更多。

## Routes 和 URL 模式
路由模式 **使用特殊语法来定义`URL`的动态参数** 并为其分配一个名字。最简单的路由没有参数：

```
local lapis = require("lapis")
local app = lapis.Application()

app:match("/", function(self) end)
app:match("/hello", function(self) end)
app:match("/users/all", function(self) end)
```

这些路由与`URL`逐字匹配。 `/` 路由是必需的。路由必须匹配请求的整个路径。这意味着对 `/hello/world` 的请求将不匹配 `/hello`。

您可以在`:`后面理解跟上一个名称来指定一个命名参数。该参数将匹配除/的所有字符（在一般情况下）：

```
app:match("/page/:page", function(self)
  print(self.params.page)
end)

app:match("/post/:post_id/:post_name", function(self) end)
```

```
在上面的例子中，我们调用 print 函数来调试，当在openresty中运行时，print的输出是被发送到nginx的notice级别的日志中去的
```
捕获的路由参数的值按其名称保存在请求对象的 `params` 字段中。命名参数必须至少包含1个字符，否则将无法匹配。

`splat`是另一种类型的模式，将尽可能匹配，包括任何`/`字符。 `splat`存储在请求对象的 `params` 表中的 `splat` 命名参数中。它只是一个单一 `*`

```
app:match("/browse/*", function(self)
  print(self.params.splat)
end)
app:match("/user/:name/file/*", function(self)
  print(self.params.name, self.params.splat)
end)
```
如果将任何文本直接放在`splat`或命名参数之后，它将不会包含在命名参数中。例如，您可以将以`.zip`结尾的网址与`/files/:filename.zip`进行匹配(那么`.zip`就不会包含在命名参数 `filename` 中)

##可选路由组件
圆括号可用于使路由的一部分可选：

```
/projects/:username(/:project)
```
以上将匹配 `/projects/leafo` 或 `/projects/leafo/lapis` 。可选组件中不匹配的任何参数在处理函数中的值将为nil。

这些可选组件可以根据需要嵌套和链接：

```
/settings(/:username(/:page))(.:format)
```

##参数字符类
字符类可以应用于命名参数，以限制可以匹配的字符。语法建模在 `Lua` 的模式字符类之后。此路由将确保该 `user_id` 命名参数只包含数字：

```
/color/:hex[a-fA-F%d]
```
这个路由只匹配十六进制参数的十六进制字符串。

```
/color/:hex[a-fA-F%d]
```

## 路由优先级
首先按优先顺序搜索路由，然后按它们定义的顺序搜索。从最高到最低的路由优先级为：

```
精确匹配的路由 /hello/world
变化参数的路由 /hello/:variable
贪婪匹配的路由 /hello/*
```

##命名路由
为您的路由命名是有用的，所以只要知道网页的名称就可以生成到其他网页的链接，而不是硬编码 `URL` 的结构。

应用程序上定义新路由的每个方法都有第二个形式，它将路由的名称作为第一个参数：

```
local lapis = require("lapis")
local app = lapis.Application()

app:match("index", "/", function(self)
  return self:url_for("user_profile", { name = "leaf" })
end)

app:match("user_profile", "/user/:name", function(self)
  return "Hello " .. self.params.name .. ", go home: " .. self:url_for("index")
end)
```

我们可以使用`self:url_for()`生成各种操作的路径。第一个参数是要调用的路由的名称，第二个可选参数是用于填充 **参数化路由** 的值的表。

点击[url_for]() 去查看不同方式去生成 `URL` 的方法。

##处理HTTP动词
根据请求的 `HTTP` 动词，进行不同的处理操作是很常见的。 `Lapis` 有一些小帮手，让写这些处理操作很简单。 `respond_to` 接收由 `HTTP` 动词索引的表，当匹配对应的动词执行相应的函数处理

```
local lapis = require("lapis")
local respond_to = require("lapis.application").respond_to
local app = lapis.Application()

app:match("create_account", "/create-account", respond_to({
  GET = function(self)
    return { render = true }
  end,
  POST = function(self)
    do_something(self.params)
    return { redirect_to = self:url_for("index") }
  end
}))
```

`respond_to` 也可以采用自己的 `before` 过滤器，它将在相应的 `HTTP` 动词操作之前运行。我们通过指定一个 `before` 函数来做到这一点。与过滤器相同的语义适用，所以如果你调用 `self:write()`，那么其余的动作将不会运行.

```
local lapis = require("lapis")
local respond_to = require("lapis.application").respond_to
local app = lapis.Application()

app:match("edit_user", "/edit-user/:id", respond_to({
  before = function(self)
    self.user = Users:find(self.params.id)
    if not self.user then
      self:write({"Not Found", status = 404})
    end
  end,
  GET = function(self)
    return "Edit account " .. self.user.name
  end,
  POST = function(self)
    self.user:update(self.params.user)
    return { redirect_to = self:url_for("index") }
  end
}))
```

在任何 `POST` 请求，无论是否使用 `respond_to`，如果 `Content-type` 头设置为 `application/x-www-form-urlencoded`，那么请求的主体将被解析，所有参数将被放入 `self.params`。

您可能还看到了 `app:get()` 和 `app:post()` 方法在前面的示例中被调用。这些都是封装了 `respond_to` 方法，可让您快速为特定 `HTTP` 动词定义操作。你会发现这些包装器最常见的动词：`get`，`post`，`delete`，`put`。对于任何其他动词，你需要使用`respond_to`。

```
app:get("/test", function(self)
  return "I only render for GET requests"
end)

app:delete("/delete-account", function(self)
  -- do something destructive
end)
```

## Before Filters
有时你想要一段代码在每个操作之前运行。一个很好的例子是设置用户会话。我们可以声明一个 before 过滤器，或者一个在每个操作之前运行的函数，像这样：

```
local app = lapis.Application()

app:before_filter(function(self)
  if self.session.user then
    self.current_user = load_user(self.session.user)
  end
end)

app:match("/", function(self)
  return "current user is: " .. tostring(self.current_user)
end)
```

你可以通过多次调用 `app:before_filter` 来随意添加。它们将按照注册的顺序运行。

如果一个 `before_filter` 调用 `self:write()`方法，那么操作将被取消。例如，如果不满足某些条件，我们可以取消操作并重定向到另一个页面：

```
local app = lapis.Application()

app:before_filter(function(self)
  if not user_meets_requirements() then
    self:write({redirect_to = self:url_for("login")})
  end
end)

app:match("login", "/login", function(self)
  -- ...
end)
```

`self:write()` 是处理一个常规动作的返回值，所以同样的事情你可以返回一个动作，可以传递给 `self:write()`


##请求对象
每个操作在调用时会请求对象作为其第一个参数传递。由于调用第一个参数 `self` 的约定，我们在一个操作的上下文中将请求对象称为 `self`。

请求对象具有以下参数：

1. `self.params`	一个包含所有`GET`，`POST` 和 `URL` 参数的表
2. `self.req`		原始请求表（从ngx状态生成）
3. `self.res` 	原始响应表（从ngx状态生成）
4. 
