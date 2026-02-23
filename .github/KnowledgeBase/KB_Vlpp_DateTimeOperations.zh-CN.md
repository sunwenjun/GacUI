# 日期与时间操作（Date and Time Operations）

跨平台日期时间处理能力，支持时区转换与时间算术。

## 概览

`DateTime` 类提供跨平台日期时间能力，包含：

- 本地时间与 UTC 时间获取
- 时区互转
- 时间加减运算
- 通过注入实现测试替身与行为定制

## 当前时间获取

- 使用 `DateTime::LocalTime()` 获取当前本地时区时间。
- 使用 `DateTime::UtcTime()` 获取当前 UTC 时间。

两者都返回 `DateTime`，表示同一“当前时刻”在不同时区语义下的表示。

## DateTime 构造

- 使用 `DateTime::FromDateTime(year, month, day, hour, minute, second, milliseconds)` 精确构造时间。
- 使用 `DateTime::FromOSInternal(osInternal)` 从操作系统内部时间表示构造。

`FromDateTime` 会按日历规则校验输入参数。

## 时区转换

- 使用 `ToLocalTime()` 将 UTC 表示转换为本地时间表示。
- 使用 `ToUtcTime()` 将本地时间表示转换为 UTC。

转换会考虑系统时区配置与夏令时规则，并返回新的 `DateTime` 对象。

## 时间算术

- 使用 `Forward(milliseconds)` 向前推进指定毫秒数。
- 使用 `Backward(milliseconds)` 向后回退指定毫秒数。

两者都返回新的 `DateTime`。

## 实现注入

可覆盖默认 DateTime 实现以支持测试与自定义行为：

- `InjectDateTimeImpl(impl)`：注入自定义 `IDateTimeImpl`。
- `EjectDateTimeImpl(impl)`：移除指定注入实现。
- `EjectDateTimeImpl(nullptr)`：移除全部注入并恢复默认 OS 实现。

注入实现形成链式结构，后注入可委托给前一个实现。

测试中请始终执行清理，避免污染其他用例。

## 额外内容

### 平台抽象

`DateTime` 屏蔽 Windows 与 Linux 底层时间 API 差异，提供一致接口。

### 精度与范围

`DateTime` 以毫秒精度进行操作，可覆盖绝大多数业务场景。

### 线程安全

`DateTime` 是不可变值类型，可安全跨线程传递。
注入机制建议在多线程运行前完成配置。

### 组件协同

`DateTime` 常与 Locale（格式化）和单元测试框架（时间相关测试）配合使用。
