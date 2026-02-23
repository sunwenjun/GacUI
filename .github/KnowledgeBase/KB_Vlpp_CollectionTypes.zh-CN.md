# 集合类型（Collection Types）

实现 `IEnumerable` 接口的动态容器系统，提供完备的数据操作能力。

## 概览

Vlpp 中每个集合类型都实现 `IEnumerable<T>`，从而统一了枚举语义。其设计风格接近 C# 集合体系，`IEnumerable<T>` / `IEnumerator<T>` 提供一致的遍历体验。

## 核心集合类型

### Array<T>

固定长度、支持随机访问。

- `Array<T>(vint size)`：创建指定大小数组，初始值可能为随机值。
- `Array<T>(T* buffer, vint size)`：从已有缓冲区复制构造。

常用操作：

- `Count()`：获取长度。
- `Get(index)` / `[index]`：按下标访问。
- `Contains(value)` / `IndexOf(value)`：查询元素。
- `Resize(size)`：调整大小并尽量保留旧值。
- `Set(index, value)` / `[index] = value`：设置元素。

### List<T>

可变长度动态数组，支持插入与删除。

常用操作：

- `Add(value)`：尾部追加。
- `Insert(index, value)`：按位置插入。
- `Remove(value)`：移除首个相等元素。
- `RemoveAt(index)` / `RemoveRange(index, count)`：按位置删除。
- `Clear()`：清空。
- 其余访问与查询操作与 `Array<T>` 类似。

### SortedList<T>

自动保持有序的列表。

- 具备 `List<T>` 大部分能力，但不支持会破坏有序性的 `Insert` / `Set`。
- `Add(value)` 会在保持有序的前提下插入。

## 关联集合

### Dictionary<Key, Value>

一对一映射，按 key 有序。

- 实现 `IEnumerable<Pair<K, V>>`。
- `Add(key, value)`：新增键值对（若 key 已存在会失败）。
- `Set(key, value)`：设置键值对（若 key 已存在则覆盖）。
- `Get(key)` / `[key]`：按 key 读取。
- `Keys()` / `Values()`：按 key 顺序获取键集合和值集合。

### Group<Key, Value>

一对多映射，按 key 有序。

- 实现 `IEnumerable<Pair<K, V>>`。
- 同一 key 可关联多个 value。
- `Get(key)` / `[key]` 返回该 key 下的全部值。
- 支持 `Contains(key)` 与 `Contains(key, value)` 查询。

## 枚举与迭代

所有集合支持范围 for：

```cpp
for (auto number : numbers)
{
    // 处理元素
}
```

可借助 `indexed(collection)` 进行“索引 + 值”联合迭代：

```cpp
for (auto [index, value] : indexed(collection))
{
    // 同时使用下标和值
}
```

## 与 LINQ 协同

通过 `From(collection)` 可转为 `LazyList<T>`，进行链式函数式处理：

```cpp
auto result = From(numbers)
    .Where([](vint x) { return x > 0; })
    .Select([](vint x) { return x * 2; })
    .ToList();
```

## 常见选型模式

- `Dictionary<K, V>`：每个 key 仅对应一个 value。
- `Group<K, V>`：一个 key 对应多个 value。
- `Array<T>`：长度固定。
- `List<T>`：频繁增删。
- `SortedList<T>`：始终保持有序。

## 额外说明

- 集合默认不保证线程安全；并发访问请配合同步原语。
- 存储引用类型时，请使用 `Ptr<T>` 管理生命周期。
- 集合支持拷贝语义：值类型会复制值，`Ptr<T>` 会共享引用。
