# 使用Lua创建Lapis应用程序

## 生成一个新项目

如果您尚未阅读，请阅读[入门指南](/docs/lapis入门.md)，了解有关创建新项目骨架的信息以及OpenResty，Nginx配置和lapis命令的详细信息。

您可以在当前目录中通过运行以下命令启动一个新的Lua项目：
```
lapis new --lua
```
默认的 `nginx.conf` 为应用程序读取一个名为 `app.lua` 的文件。`lapis new` 提供了一个最基本的结构。

`app.lua` 是包含应用程序的常规 `Lua` 模块。你甚至可以像普通 `Lua` 解释器中的任何其他模块一样加载该模块。它看起来像这样：

```
-- app.lua
local lapis = require("lapis")
local app = lapis.Application()

app:get("/", function()
  return "Welcome to Lapis " .. require("lapis.version")
end)

return app
```

启动服务器
`lapis server`

访问`http://localhost:8080`以查看该页面

如果要更改端口，我们可以创建一个配置。新建`config.lua`文件。

在本例中，我们将开发环境中的端口更改为`9090`：

```
-- config.lua
local config = require("lapis.config")

config("development", {
  port = 9090
})
```
您可以在[配置和环境指南]()中阅读有关配置的更多信息。


当运行 `lapis server` 而没有其他参数时，会自动使用和加载`development`环境。 （而且文件`lapis_environment.lua`不存在）

`Lapis` 在配置中使用少量的字段（如 `port` ），其他字段可以用来存储任何你想要的。例如：

```
-- config.lua
local config = require("lapis.config")

config("development", {
  greeting = "Hello world"
})
```

您可以通过调用 `get` 获取当前配置。它返回一个简单的 `Lua` 表：

```
-- app.lua
local lapis = require("lapis")
local config = require("lapis.config").get()

local app = lapis.Application()

app:get("/", function(self)
  return config.greeting .. " from port " .. config.port
end)

return app
```

## 创建view
现在我们可以创建基本的页面，我们可能会想要一些更复杂的东西。 `Lapis` 支持`etlua`，一种 `Lua` 模板语言，允许您在 `html` 和 文本中插入 `Lua` 代码

视图是负责生成 `HTML` 的文件。通常，您的操作将准备视图的所有数据，然后指示它进行渲染。

默认情况下，`Lapis` 在 `views/` 目录中搜索视图。让我们在那里创建一个新的视图文件`index.etlua`。我们不会使用任何 `etlua` 的特殊标记，所以它看起来像一个正常的HTML文件。

```
<!-- views/index.etlua -->
<h1>Hello world</h1>
<p>Welcome to my page</p>
```
你会注意到，<html>，<head>和<body>标签不在那里。视图通常呈现页面的内部，并且布局负责其周围的东西。我们将进一步看看布局。

现在让我们创建呈现我们的视图的应用程序：

```
-- app.lua
local lapis = require("lapis")

local app = lapis.Application()
app:enable("etlua")

app:get("/", function(self)
  return { render = "index" }
end)

return app
```
`etlua` 在默认情况下不启用，您必须通过调用应用程序实例上的 `enable` 方法来启用它。

我们使用返回值的 `render` 参数来指示在呈现页面时要使用的模板。在这种情况下，`index` 是指名为 `views.index` 的模块。 `etalua` 将自己注入到 `Lua`的`require` 方法中，因此当模块 `views.index` 被加载时，会尝试读取和解析文件`views/index.etlua` 。

运行服务器并在浏览器中导航到它应该可以显示出我们渲染的模板。

##使用etlua
`etlua` 提供了以下标签用于将 `Lua` 代码注入到模板中：

```
<％lua_code％> 运行的Lua代码块
<％= lua_expression％> 将表达式的结果写入输出，HTML转义
<％- lua_expression％> 同上，但没有HTML转义
```
在 [etlua指南]() 中了解有关 `etlua` 集成的更多信息。
在下面的示例中，我们在操作中分配一些数据，然后在我们的视图中打印出来：

```
-- app.lua
local lapis = require("lapis")

local app = lapis.Application()
app:enable("etlua")

app:get("/", function(self)
  self.my_favorite_things = {
    "Cats",
    "Horses",
    "Skateboards"
  }

  return { render = "list" }
end)

return app
```

```
<!-- views/list.etlua -->
<h1>Here are my favorite things</h1>
<ol>
  <% for i, thing in pairs(my_favorite_things) do %>
    <li><%= thing %></li>
  <% end %>
</ol>
```

## 创建布局
布局是一个单独的共享模板，每个页面都包含的内容。 `Lapis` 有一个基本的布局让你使用，但你很可能想替换它的东西而自定义。

我们将在 `etlua` 中编写布局，就像我们的视图。创建 `view/layout.etlua`：

```
<!-- views/layout.etlua -->
<!DOCTYPE HTML>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title><%= page_title or "My Page" %></title>
</head>
<body>
  <h1>Greetings</h1>
  <% content_for("inner") %>
</body>
</html>
```

`content_for` 函数是模板中内置的特殊函数，它允许您 **将数据从视图发送到布局**。 `Lapis` 将视图的渲染结果放入名为 `inner` 的内容变量。你会注意到，我们不需要使用任何写入页面的 `etlua` 标签。这是因为 `content_for` 有效地将其结果直接放入输出缓冲区。

通常在视图中可用的任何其他变量和帮助函数也可在布局中使用。

现在布局被编写，它可以被分配给应用程序：

```
local app = lapis.Application()
app:enable("etlua")
app.layout = require "views.layout"

-- the rest of the application...
```

语法与渲染视图略有不同。不是为 `layout` 字段分配模板名称，而是分配实际的模板对象。这可以通过引入 `views.layout` 获得。如上所述，`etlua` 负责将 `.etlua` 文件转换为Lua可用的东西。

##下一个
请阅读[请求和操作]()指南，了解 `Lapis` 如何路由 `HTTP` 请求，并让您对其进行响应。
