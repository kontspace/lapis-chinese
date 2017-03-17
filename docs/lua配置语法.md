# Lua 配置语法

## 配置示例
`Lapis` 的配置模块提供了对递归合并 `table` 的支持。

例如，我们可以定义一个基本配置，然后覆盖更多具体的配置声明中的一些值：

```
-- config.lua
local config = require("lapis.config")

config({"development", "production"}, {
  host = "example.com",
  email_enabled = false,
  postgres = {
    host = "localhost",
    port = "5432",
    database = "my_app"
  }
})

config("production", {
  email_enabled = true,
  postgres = {
    database = "my_app_prod"
  }
})
```

这将产生以下两个配置结果（默认值省略）：

```
-- "development"
{
  host = "example.com",
  email_enabled = false,
  postgres = {
    host = "localhost",
    port = "5432",
    database = "my_app",
  },
  _name = "development"
}
```

```
-- "production"

{
  host = "example.com",
  email_enabled = true,
  postgres = {
    host = "localhost",
    port = "5432",
    database = "my_app_prod"
  },
  _name = "production"
}
```

您可以在相同的配置名称上调用 `config` 函数多次，每次将传入的表合并到配置中。

