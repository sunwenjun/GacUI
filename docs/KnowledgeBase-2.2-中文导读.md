# KnowledgeBase 2.2 中文导读：技术决策知识底座

## 本次处理范围

入口文档：`.github/KnowledgeBase/Index.md`

已翻译并落地到对应目录（`.github/KnowledgeBase`）：

- `KB_Vlpp_PrimitiveTypes.zh-CN.md`（类型系统 / 基础值类型）
- `KB_VlppReflection_TypeMetadata.zh-CN.md`（反射 / 类型元数据）
- `KB_VlppRegex_PatternMatching.zh-CN.md`（正则 / 模式匹配）
- `KB_Vlpp_DateTimeOperations.zh-CN.md`（时间 / DateTime）
- `KB_Vlpp_CollectionTypes.zh-CN.md`（集合 / 容器系统）

## 这些文档分别讲了什么

### 1) 类型系统：Primitive Types

核心在“表达力 + 类型安全”：

- `Nullable<T>`：在值类型上补齐空值语义。
- `Pair / Tuple`：减少样板结构体定义，配合结构化绑定提升可读性。
- `Variant<T...>`：显式建模“多类型之一”，并通过 `Apply` / `Overloading` 做类型分派。

适合场景：配置值、多形态返回值、领域模型中临时聚合结构。

### 2) 反射：Type Metadata

核心在“运行时可观测 + 动态能力”：

- 通过 `GetTypeDescriptor<T>` 获取元数据。
- 使用 `Value` 作为统一装箱容器。
- 通过 `Description<T>` / `AggregatableDescription<T>` 建立可反射类型层次。

适合场景：脚本桥接、动态调用、通用序列化和编辑器类工具。

### 3) 正则：Pattern Matching

核心在“多编码匹配 + API 分层”：

- `Match*` 返回详细匹配信息。
- `Test*` 返回布尔结果，更轻量。
- `Search / Split / Cut` 覆盖全量匹配、分割和完整切分。

并且强调了 VlppRegex 与常见正则（如 .NET）在点号语义和转义方式上的差异。

### 4) 时间：DateTime Operations

核心在“跨平台一致性 + 可测试性”：

- `LocalTime/UtcTime`、`ToLocalTime/ToUtcTime` 形成时区闭环。
- `Forward/Backward` 提供毫秒级时间算术。
- `InjectDateTimeImpl/EjectDateTimeImpl` 支持时间实现注入，方便测试。

### 5) 集合：Collection Types

核心在“容器统一接口 + 选型清晰”：

- 顺序容器：`Array`、`List`、`SortedList`
- 关联容器：`Dictionary`、`Group`
- 统一遍历接口：`IEnumerable<T>`，可无缝衔接 `From(...).Where(...).Select(...)`

## 可以学到哪些技巧（可直接落地）

1. **先定“语义类型”再写逻辑**
   - 可空值用 `Nullable<T>`，多态值用 `Variant<T...>`，避免“魔法值 + if-else 地狱”。

2. **用结构化绑定提升可读性**
   - `Pair/Tuple/indexed` 组合可显著降低样板代码。

3. **把“动态能力”封装在反射边界内**
   - 业务核心保持静态类型，动态部分用 `Value + TypeDescriptor` 集中处理。

4. **按目标选择正则 API，而不是“一把梭”**
   - 只判断真假就用 `Test`，需要结果明细才用 `Match/Search`。

5. **时间相关逻辑必须可注入、可回放**
   - 所有“当前时间”依赖都通过 `DateTime` 注入实现，测试可重复。

6. **集合选型先看约束再看习惯**
   - 固定长度选 `Array`，高频增删选 `List`，天然有序选 `SortedList`，一对多映射选 `Group`。

7. **默认走统一枚举接口，减少容器耦合**
   - 让处理逻辑依赖 `IEnumerable<T>`，而不是具体容器类型，便于重构与替换。

## 推荐阅读顺序

1. `Index.md`（先看全局 API 选择原则）
2. Primitive Types（先统一类型表达）
3. Collection Types（再定数据组织）
4. DateTime（补齐跨平台时间策略）
5. Regex（处理文本规则）
6. Reflection（最后引入动态能力）

