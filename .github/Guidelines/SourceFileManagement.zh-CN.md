# 解决方案与项目文件结构

- 一个解决方案文件（`*.sln` 或 `*.slnx`）包含多个项目文件。
- 常见的 C++ 项目文件是 `*.vcxproj` 或 `*.vcxitems` 命名的 XML 文件。
- `*.vcxitems.filters` 或 `*.vcxproj.filters` 这类 XML 文件用于组织“解决方案资源管理器文件夹（虚拟文件夹）”，它可以与物理文件系统结构不同，以提供更友好的人工视图。
- `*.vcxproj.user` 文件包含项目的某些临时本地配置。该文件不受 git 跟踪，但包含运行项目所需参数。
- 当你把源文件添加到某个“解决方案资源管理器文件夹”时：
  - 该文件也必须被加入一个或多个项目文件。
  - 找到同名的 `*.vcxitems.filters` 或 `*.vcxproj.filters` 文件。
  - 每个文件都必须挂到一个“解决方案资源管理器文件夹”下，该信息在 XPath `/Project/ItemGroup/ClCompile@Include="PhysicalFile"/Filter` 中描述。
  - `Filter` 标签内即为“解决方案资源管理器文件夹”路径。
  - 编辑对应 `*.vcxitems.filters` 或 `*.vcxproj.filters` 文件，把源文件纳入其中。

## 重命名和删除源文件

- 必须同步更新所有受影响的 `*.vcxitems`、`*.vcxproj`、`*.vcxitems.filters` 和 `*.vcxproj.filters`。
