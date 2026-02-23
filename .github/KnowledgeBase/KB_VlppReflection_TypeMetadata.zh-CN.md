# 类型元数据访问（Type Metadata Access）

## 通过反射系统在运行时获取与操作类型信息

当启用反射（未定义 `VCZH_DEBUG_NO_REFLECTION`）时，可访问类型元数据并执行动态操作。该反射系统提供完整的运行时类型信息，适用于动态编程场景。

## 核心 API

### GetTypeDescriptor

- 在启用反射时，使用 `vl::reflection::description::GetTypeDescriptor<T>` 获取类型元数据。
- 返回 `ITypeDescriptor*`，包含类型的完整描述信息。
- 仅在编译包含反射能力时可用（非 `VCZH_DEBUG_NO_REFLECTION` 模式）。

### Value 装箱

- 使用 `vl::reflection::description::Value` 对任意值类型进行装箱，类似 C# 的 `object` 或 C++ 的 `std::any`。
- 提供类型安全的任意值存取能力。
- 支持兼容类型之间的转换。
- 可作为任意可反射类型的统一容器。

### 可反射基类

- 使用 `Description<T>` 使类具备反射能力。
- 使用 `AggregatableDescription<T>` 使类可在 Workflow 脚本中继续被继承。
- 对于无其他父接口的可反射接口，使用 `IDescriptable`。

## 类型层次要求

可反射类必须继承 `public vl::reflection::Description<类自身>`。
若希望该类可在 Workflow 脚本中被继承，应改用 `AggregatableDescription`。

可反射类或接口的子类型不会自动成为可反射类型，仍需显式继承 `Description<T>` 或 `AggregatableDescription<T>`。

可反射接口同样需要继承 `public vl::reflection::Description<接口自身>`。
如果该接口不继承其他接口，还必须继承 `public vl::reflection::IDescriptable`。

## 使用模式

当访问可反射成员或函数时，`vl::reflection::description::Value` 很有帮助。
它是值类型，可作为“任意类型值”容器，行为类似 C#/Java 的 `object` 或 C++ 的 `std::any`。

当 `Value` 仅用于表达“任何值”时，不一定依赖反射开启。

某类型是否“可反射”，取决于是否通过反射注册机制完成注册。

## 额外内容

### 条件使用

请先判断反射是否可用，再调用反射 API：

```cpp
#ifndef VCZH_DEBUG_NO_REFLECTION
    auto typeDescriptor = vl::reflection::description::GetTypeDescriptor<MyClass>();
    if (typeDescriptor)
    {
        // 使用类型描述符读取元数据
    }
#endif
```

### Value 类型能力

`Value` 支持：

- 任意可反射类型的自动装箱 / 拆箱
- 类型检查与安全转换
- 与反射系统其他能力联动
- 序列化与反序列化场景

### 类型安全

反射系统在提供动态能力的同时保持 C++ 类型安全：

- 尽可能保留编译期类型检查
- 动态操作时执行运行时类型验证
- 类型不匹配时使用异常机制报告错误
