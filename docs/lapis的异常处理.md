# lapis的异常处理

## 错误的种类
`Lapis` 区分两种错误：可恢复和不可恢复错误。 `Lua` 的运行时在执行期间抛出的错误或调用错误被认为是不可恢复的。 （这也包括 `Lua` 内置函数 `assert` ）

因为不可恢复的错误不会被用户捕获，所以 `Lapis` 捕获它们并向浏览器打印一个异常消息。任何已经运行的操作都可能会被中止，`Lapis` 将打印一个特殊视图来显示堆栈以及将 `status` 设置为 `500`。

这些类型的错误通常是一个 `bug` 或其他严重的问题，并且应该被修复。

可恢复的错误是用户控制中止执行一个处理函数以运行指定错误处理函数的方式。它们使用协程而不是 `Lua` 的错误系统来实现。

比如来自用户的无效输入或数据库中缺少的记录。

## 捕获可恢复的错误
`capture_errors` 帮助程序用于包装一个操作，以便它可以捕获错误并运行错误处理程序。

它不捕获运行时错误。如果你想捕获运行时错误，你应该使用 `pcall`，就像你通常在 `Lua`中做的那样。

`Lua` 没有大多数其他语言的异常概念。相反，`Lapis`使用协同程序创建一个异常处理系统。我们使用 `capture_errors` 帮助程序来定义我们必须捕获错误的范围。然后我们可以使用 `yield_error` 来抛出一个原始错误。

```
local lapis = require("lapis")
local app_helpers = require("lapis.application")

local capture_errors, yield_error = app_helpers.capture_errors, app_helpers.yield_error

local app = lapis.Application()

app:match("/do_something", capture_errors(function(self)
  yield_error("something bad happened")
  return "Hello!"
end))
```

当出现错误时会发生什么？该操作将在第一个错误处停止执行，然后运行错误处理程序。默认错误处理程序将在 `self.errors` 中设置一个类似数组的表，并返回 `{render = true}`。在您的视图中，您可以显示这些错误消息。这意味着如果你有一个命名的路由，那个路由的视图将会被渲染。然后当出现一个 `error` 表时你应该编写你自己的视图。

如果你想有一个自定义的错误处理程序，你可以传入一个 `table` 来调用`capture_errors` ：（注意 `self.errors` 在自定义处理程序之前设置）

```
app:match("/do_something", capture_errors({
  on_error = function(self)
    log_erorrs(self.errors) -- you would supply the log_errors function
    return { render = "my_error_page", status = 500 }
  end,
  function(self)
    if self.params.bad_thing then
      yield_error("something bad happened")
    end
    return { render = true }
  end
}))
```

当调用 `capture_errors`处理函数时将使用传入的 `table` 的第一个位置值作为操作。

如果您正在构建 `JSON API`，`lapis` 则会提供另一个方法`capture_errors_json`，它会在 `JSON` 对象中呈现错误，如下所示：

```
local lapis = require("lapis")
local app_helpers = require("lapis.application")

local capture_errors_json, yield_error = app_helpers.capture_errors_json, app_helpers.yield_error

local app = lapis.Application()

app:match("/", capture_errors_json(function(self)
  yield_error("something bad happened")
end))
```

然后将呈现如下错误（请使用正确的`content-type`）

```
{ errors: ["something bad happened"] }
```

### `assert_error`
在 `lua` 中，当一个函数执行失败时，习惯返回 `nil` 和一个错误消息。为此，提供了一个 `assert_error` 帮助程序。如果第一个参数是 `falsey`（ `nil` 或 `false`），那么第二个参数作为一个错误被抛出，否则所有的参数都会从函数返回。

使用 `assert_error` 对于数据库的方法非常方便。

```
local lapis = require("lapis")
local app_helpers = require("lapis.application")

local capture_errors, assert_error = app_helpers.capture_errors, app_helpers.assert_error

local app = lapis.Application()

app:match("/", capture_errors(function(self)
  local user = assert_error(Users:find({id = "leafo"}))
  return "result: " .. user.id
end))
```
