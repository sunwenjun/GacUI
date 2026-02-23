# Vlpp 基础值类型（Primitive Value Types）

用于在 Vlpp 库中组织和操作相关数据值的容器类型。

## Nullable<T>

`Nullable<T>` 为类型 `T` 增加了 `nullptr` 语义。`Nullable<T>` 可以直接被赋值为 `T`，此时对象变为“非空”；否则为“空”。

- 使用 `Reset()` 将 `Nullable<T>` 重置为空。
- 使用 `operator bool` 判断是否为空。
- 仅在确认非空时调用 `Value()` 获取内部值。

`Value()` 返回只读值：你不能直接修改内部值本身，但可以将整个 `Nullable<T>` 重新赋值。

同类型 `Nullable<T>` 之间可以按标准 C++ 方式比较。

### 使用示例

```cpp
// 创建可空整数
Nullable<vint> nullableInt;
if (!nullableInt) {
    // nullableInt 当前为空
}

// 赋值
nullableInt = 42;
if (nullableInt) {
    vint value = nullableInt.Value(); // value == 42
}

// 重置为空
nullableInt.Reset();
```

## Pair<Key, Value>

`Pair<K, V>` 用于存放一个 `K` 类型的 `key` 和一个 `V` 类型的 `value`，是表示二元组的简便结构。

- 使用字段 `key`、`value` 访问成员。
- 使用 `Pair(k, v)` 可进行类型推导，提升可读性。

同类型 `Pair<K, V>` 之间可以按标准 C++ 方式比较。

`Pair<K, V>` 支持结构化绑定。

### 使用示例

```cpp
// 利用类型推导创建 Pair
auto pair = Pair(L"name", 25);

// 访问字段
WString name = pair.key;
vint age = pair.value;

// 结构化绑定
auto [personName, personAge] = pair;

// 比较
Pair<WString, vint> pair1(L"Alice", 30);
Pair<WString, vint> pair2(L"Bob", 25);
bool isEqual = (pair1 == pair2); // false
```

### 与集合结合的结构化绑定

你可以使用 `indexed` 将 `IEnumerable<T>` 转换为 `IEnumerable<Pair<vint, T>>`，以便通过 `for(auto [index, x] : indexed(xs))` 同时获取索引和值。

```cpp
List<WString> names;
names.Add(L"Alice");
names.Add(L"Bob");

for(auto [index, name] : indexed(names)) {
    Console::WriteLine(itow(index) + L": " + name);
}
```

## Tuple<T...>

`Tuple<...>` 可在不定义 `struct` 的情况下组织多个值。

- 使用 `Tuple(a, b, c...)` 可自动推导类型。
- Tuple 内部值不可原位修改，但可整体赋值为另一个 tuple。
- 使用 `get<0>()` 访问第一个元素，索引必须是编译期常量。

同类型 `Tuple<...>` 之间可以按标准 C++ 方式比较。

`Tuple<...>` 支持结构化绑定。

### 使用示例

```cpp
// 类型推导创建 tuple
auto tuple = Tuple(L"Alice", 25, true);

// 通过 get<index>() 访问
WString name = tuple.get<0>();
vint age = tuple.get<1>();
bool isActive = tuple.get<2>();

// 结构化绑定
auto [personName, personAge, active] = tuple;

// 比较
auto tuple1 = Tuple(L"Alice", 25, true);
auto tuple2 = Tuple(L"Bob", 30, false);
bool isEqual = (tuple1 == tuple2); // false
```

## Variant<T...>

`Variant<T...>` 表示“多个类型之一”，但任一时刻只能存一种值。

- `Variant<T...>` 必须初始化或赋值，不能“空”。
- 若需要可空语义，推荐在类型列表中加入 `nullptr_t`，而不是 `Nullable<Variant<...>>`。
- 使用 `Index()` 获取当前存储类型的序号（从 0 开始）。
- 在确认类型时，用 `Get<T>()` 获取值。
- 不确定类型时，用 `TryGet<T>()` 获取指针；不匹配返回 `nullptr`。
- 使用 `Apply` + 回调（通常是模板 lambda）统一处理多类型。
- 使用 `Overloading` 与 `Apply` 组合，可按类型分派。
- `TryApply` 类似 `Apply`，但不要求覆盖所有类型分支。

### 使用示例

```cpp
Variant<WString, vint, bool> value = L"Hello";

vint typeIndex = value.Index(); // 0:WString, 1:vint, 2:bool

if (value.Index() == 0) {
    WString str = value.Get<WString>();
}

if (auto ptr = value.TryGet<WString>()) {
    WString str = *ptr;
}

value.Apply(Overloading(
    [](WString& str) { Console::WriteLine(L"String: " + str); },
    [](const vint& num) { Console::WriteLine(L"Number: " + itow(num)); },
    [](bool flag) { Console::WriteLine(flag ? L"True" : L"False"); }
));
```

### 可空 Variant

如果需要“可空 variant”，请在类型列表中包含 `nullptr_t`：

```cpp
Variant<WString, vint, bool, nullptr_t> nullableValue = nullptr;
```
