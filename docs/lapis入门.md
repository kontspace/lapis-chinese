# lapis入门
Lapis 是为 Lua 和 MoonScript 编写的 Web 框架。 
Lapis 很有趣，因为它建立在Nginx 发行的 OpenResty 之上。
您的 Web 应用程序直接在 Nginx 内部运行。 

Nginx 的事件循环允许您使用 OpenResty 提供的模块进行异步 HTTP 请求，数据库查询和其他请求。 
Lua 的协程允许你编写在后台事件驱动的同步代码。
除了提供Web框架，Lapis还提供了用于在不同配置环境中控制 OpenResty 的工具。
即使你不想使用 Web 框架，但如果你使用 OpenResty，你也许会发现它依旧是是有用的。
 
这个 Web 框架实现了 URL 路由器，HTML 模板，CSRF 和会话支持，PostgreSQL 或 MySQL 支持的主动记录系统，用于处理 model 和开发网站所需的一些其他有用的功能。

本指南希望能够给大家作为一个教程和参考

## 基本设置

将 OpenResty 安装到系统上。如果你使用 Heroku，那么你可以使用 Heroku OpenResty 模块和 Lua 构建包。

### 使用luarocks来安装lapis

`luarocks install lapis`

## 创建一个应用
lapis 命令行工具

Lapis 附带了一个命令行工具，可帮助您创建新项目和启动服务器。要看看Lapis能做什么，在你的 shell 中运行 `lapis help`。

现在，我们将创建一个新项目。切换到一个干净的目录并运行如下命令：

```shell
lua new
wrote   nginx.conf
wrote   mime.types
wrote   app.moon
```

Lapis编写一个基本的 Nginx 配置和一个空白 Lapis 应用程序。

随意查看生成的配置文件（nginx.conf是唯一重要的文件）。

以下是它的功能的简要概述：

>任何请求在 `/static/` 将提供静态文件（如果你要提供这个功能，你可以创建这个目录）对 `/favicon.ico` 的请求则响应，
>`/static/favicon.ico` 这个文件然后所有其他请求将由 Lua 提供，更具体地说是一个名为 `app` 的模块。

当您使用 lapis 命令行工具启动服务器时，将处理 `nginx.conf` 文件，并使用当前 lapis 环境中的值填充模板变量，这将在下面更详细地讨论。

## nginx 配置
让我们来看看 `lapis new` 给我们的配置。
虽然没有必要立即查看，但如果想要构建更高级的应用程序或者甚至只是想将应用程序部署到生产环境，那么了解它是很重要的。

这里是生成的 `nginx.conf`：

```nginx
worker_processes ${{NUM_WORKERS}};
error_log stderr notice;
daemon off;

events {
  worker_connections 1024;
}

http {
  include mime.types;

  server {
    listen ${{PORT}};
    lua_code_cache ${{CODE_CACHE}};

    location / {
      default_type text/html;
      content_by_lua '
        require("lapis").serve("app")
      ';
    }

    location /static/ {
      alias static/;
    }

    location /favicon.ico {
      alias static/favicon.ico;
    }
  }
}
```

首先要注意的是，这不是一个正常的 Nginx 配置文件。lapis 使用特殊的 `${{VARIABLE}}` 语法在启动服务器之前注入环境设置。

`error_log stderr notice` 和 `daemon off` 让我们的服务器在前台运行，并将日志打印到控制台。
这对于开发时是很好的，但是在生产时一定要关闭 `lua_code_cache` 对于开发时也是另一个有用的设置。
当设置为 `off` 时，将导致所有`Lua` 模块在每个请求时重新加载。

对 Web 应用程序的源代码的修改可以自动重载。在生产环境中，应当启用（`on`）缓存以获得最佳性能。默认为`off`。

`content_by_lua` 指令指定一个 `Lua` 代码块，它将处理与其他 `location` 不匹配的任何请求。
它加载 `Lapis` 并告诉它为名为 `app` 的模块提供服务。之前运行的 `lapis new` 提供了一个框架模块`app`来开始。

## 启动服务器
虽然可以手动启动 `Nginx` ，但是 `Lapis` 包装了一个方便的命令去构建配置和启动服务器。

在 `shell` 中运行 `lapis server` 将启动服务器。 `lapis` 将尝试查找您的OpenResty安装。
它将搜索以下目录中的nginx二进制文件。 （最后一个代表你PATH中的任何东西）

```shell
"/usr/local/openresty/nginx/sbin/"
"/usr/local/opt/openresty/bin/"
"/usr/sbin/"
""
```


```
记住，你需要 OpenResty 而不是正常安装 Nginx。 Lapis 将忽略常规的 Nginx 二进制文件。
```

请继续并启动服务器，看看它是什么样子： `lapis server`

默认配置将服务器置于前台运行，使用CTRL + C停止服务器。
如果服务器在后台运行，可以使用 `lapis term` 停止。
它必须在应用程序的根目录中运行。此命令查找正在运行的服务器的PID文件，并向该进程发送 `TERM` 消息（如果存在）


## 创建一个应用
现在你知道如何生成一个新项目并启动和停止服务器，你已经准备好开始编写应用程序代码。本指南分为 `MoonScript` 和 `Lua` 两个。

如果你不确定要使用什么，我建议通过两个路径阅读。

- [Create an application with Lua](http://leafo.net/lapis/reference/lua_getting_started.html)
- [Create an application with MoonScript](http://leafo.net/lapis/reference/moon_getting_started.html)
