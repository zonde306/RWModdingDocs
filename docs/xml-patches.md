# XML Patch：XPath、操作与兼容性

## 执行模型

`Patches/**/*.xml` 使用 `<Patch>` 根节点。加载器先合并各 mod 的原始 Def XML，再按 mod 顺序执行 Patch，随后才展开继承、实例化 Def。因此 XPath 匹配的是 **原始合并 DOM**，不是继承完成的对象，也不是游戏中调试显示的最终属性。

```xml
<?xml version="1.0" encoding="utf-8"?>
<Patch>
  <Operation Class="PatchOperationReplace">
    <xpath>Defs/ThingDef[defName="Steel"]/statBases/MarketValue</xpath>
    <value><MarketValue>2</MarketValue></value>
  </Operation>
</Patch>
```

示例 XPath 已在本地 Core XML 验证目标；价格仅作演示。`value` 包含要放进 DOM 的节点本身，不是只写裸数字。

## 已核实的原版操作

| Class | 字段 | 行为 / 失败语义 |
|---|---|---|
| `PatchOperationAdd` | xpath、value、可选 order | 向所有命中节点追加 value 子节点；`order=Prepend` 可前插；零命中返回 false |
| `PatchOperationReplace` | xpath、value | 对每个命中节点插入替代节点并移除原节点；零命中返回 false；不会创建缺失路径 |
| `PatchOperationRemove` | xpath | 删除全部命中节点；零命中返回 false |
| `PatchOperationTest` | xpath | 存在匹配节点才返回 true；不修改 DOM |
| `PatchOperationConditional` | xpath、match、nomatch | 根据节点存在性选择操作；只提供所需分支即可；分支执行结果向外传播 |
| `PatchOperationSequence` | operations 列表 | 顺序执行，遇到 false 就停止；**不回滚**已经完成的操作 |
| `PatchOperationFindMod` | mods、match、nomatch | 按激活 mod 的**显示名称**检测，任一名称命中即 match；不是 packageId |

以上行为核对了对应 `ApplyWorker`。其他操作（如插入兄弟节点、属性修改）先查 `Verse.PatchOperation*`，不要从框架 mod 或旧教程猜字段。私有字段也可能被 XML 加载；Patch 类的实际字段定义才是格式依据。

## 常用 XPath

```text
Defs/ThingDef[defName="EX17_Target"]
Defs/ThingDef[@Name="BuildingBase"]
Defs/ThingDef[@ParentName="FurnitureBase"]
Defs/ThingDef[defName="EX17_Target"]/comps/li[@Class="Example17.CompProperties_Counter"]
Defs/RecipeDef[defName="EX17_Recipe"]/recipeUsers/li[text()="CraftingSpot"]
```

- `@Name` / `@ParentName` 选择属性，`defName` 选择子元素。不要把 defName 写成 `@defName`。
- 选子模板时无法匹配仅存在于父模板的字段。改父模板影响全部子孙；只改子 Def 时，向子节点补入覆盖字段。
- `//ThingDef` 或 `contains(...)` 容易扩大修改范围；优先精确 defName。每条补丁明确预期匹配数量。
- 用命名空间限定的自定义 Def 标签时，标签可能包含点；按目标 XML 实际写法匹配。
- XPath 字符串中的 `<` / `&` 需要 XML 转义；不要随意把游戏 Def 元素放进 XML namespace。

## 路径存在时替换，否则新增

以下例子同时处理字段存在和只从父模板继承的情况。必须先保证 `EX17_Counter` 存在；示例配套定义见 [recipes.md](recipes.md)。

```xml
<Patch>
  <Operation Class="PatchOperationConditional">
    <xpath>Defs/ThingDef[defName="EX17_Counter"]/stackLimit</xpath>
    <match Class="PatchOperationReplace">
      <xpath>Defs/ThingDef[defName="EX17_Counter"]/stackLimit</xpath>
      <value><stackLimit>1</stackLimit></value>
    </match>
    <nomatch Class="PatchOperationAdd">
      <xpath>Defs/ThingDef[defName="EX17_Counter"]</xpath>
      <value><stackLimit>1</stackLimit></value>
    </nomatch>
  </Operation>
</Patch>
```

处理缺失的 `<comps>`、`<statBases>` 等容器也一样：先检查容器，已有时向容器添加项目，缺失时向 Def 加整个容器。往已有容器旁追加另一个同名容器不是合并。

## 多步操作与可选兼容

```xml
<Patch>
  <Operation Class="PatchOperationSequence">
    <operations>
      <li Class="PatchOperationTest">
        <xpath>Defs/ThingDef[defName="Steel"]</xpath>
      </li>
      <li Class="PatchOperationReplace">
        <xpath>Defs/ThingDef[defName="Steel"]/statBases/MarketValue</xpath>
        <value><MarketValue>2</MarketValue></value>
      </li>
    </operations>
  </Operation>
</Patch>
```

不要给 Sequence 填一个未经核实的 `success` 字段来吞失败；本地类只有 operations 等实际字段。先完成的步骤即使后续失败也会留下效果。

可选 mod 的补丁优先放 `LoadFolders.xml` 的条件目录，使用稳定 packageId；见 [mod-layout.md](mod-layout.md)。确需 FindMod 时填写 About 的显示名称，并说明改名风险。XML MayRequire 的处理时机与 Patch 选择不是一回事，不要在 `<Operation>` 上凭空加条件属性期待自动跳过。

## 有效验证

1. XML 能被 `XmlDocument` 解析。
2. 在原始合并 DOM 中执行 XPath，记录预期/实际命中数量；只检 Core 文件不能验证第三方合并结果。
3. 理解父链，检查补丁有没有造成重复字段、重复 Comp 或意外修改所有子类。
4. 以目标依赖顺序启动，确认没有 Patch 失败、字段加载或跨引用红字。
5. 检查游戏内最终数值；第三方 Patch、StatPart、stuff、品质、hediff 可能继续影响显示值。

离线 XPath 检查与日志入口见 [debugging.md](debugging.md)。只有 XML 语法检查通过时，交付必须写“未验证游戏加载/最终行为”。
