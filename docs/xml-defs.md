# Def XML：格式、继承与常用定义

## 格式模型

Def XML 没有覆盖所有 mod 类型的固定 XSD。真正的 schema 是 **当前 C# 类型的可加载字段 + 原版 XML 惯例 + 自定义加载器**。XML 合法只表示语法通过，不代表字段、Def 引用或业务配置正确。

```xml
<?xml version="1.0" encoding="utf-8"?>
<Defs>
  <ThingDef Name="EX17_ItemBase" ParentName="ResourceBase" Abstract="True">
    <stackLimit>75</stackLimit>
  </ThingDef>
  <ThingDef ParentName="EX17_ItemBase">
    <defName>EX17_Component</defName>
    <label>example component</label>
    <description>A component used by the example mod.</description>
    <graphicData>
      <texPath>Things/Item/Resource/Steel</texPath>
      <graphicClass>Graphic_StackCount</graphicClass>
    </graphicData>
    <statBases>
      <MarketValue>5</MarketValue>
      <Mass>0.2</Mass>
    </statBases>
  </ThingDef>
</Defs>
```

示例复用原版资源，不需要新纹理。`ResourceBase` 的实际字段位于 `Data/Core/Defs/ThingDefs_Items/Items_Resource_Base.xml`。复制一个父模板时，必须检查它带入的类别、ticker、comps、statBases 等。

## 标识符与属性

| 名称 | 含义 | 示例 |
|---|---|---|
| `defName` 子元素 | 运行时 Def 标识；引用和存档都依赖它 | `EX17_Component` |
| `Name` 属性 | XML 模板名；不是运行时 Def 引用 | `EX17_ItemBase` |
| `ParentName` 属性 | 继承 `Name` 对应模板 | `ResourceBase` |
| `Abstract="True"` | 模板不实例化为 Def | 可不写 defName |
| `Class` 属性 | 该 XML 节点要构造的 C# 类型 | `Example17.CompProperties_Counter` |
| `thingClass` 子元素 | ThingDef 生成的实体类 | `Building` / 自定义 Thing 子类 |
| `compClass` 子元素 | CompProperties 对应的运行时 Comp 类 | 通常在 C# 构造函数中赋值 |
| `MayRequire` 属性 | 指定包全部启用时才加载该节点 | `Ludeon.RimWorld.Biotech` |
| `MayRequireAnyOf` 属性 | 指定包任一启用时才加载该节点 | 逗号分隔包 ID |
| `Inherit="False"` | 在该子树上停止合并父节点内容 | `<comps Inherit="False">...` |

Def 名、字段名、类型名和路径保留原有大小写。对自己新增的 Def、模板、key 使用统一前缀，避免全局冲突。已有 defName 不要随意改名，否则旧存档引用可能丢失。

## 值如何映射到 C#

| C# 字段种类 | XML 形式 | 备注 |
|---|---|---|
| string / bool / int / float | `<stackLimit>75</stackLimit>`、`<rotatable>false</rotatable>` | 数字使用点作小数点；不要按系统区域写逗号小数 |
| enum | `<tickerType>Rare</tickerType>` | 必须是当前枚举成员 |
| Def 引用 | `<designationCategory>Misc</designationCategory>` | 内容是目标 defName，不是 label |
| Type 引用 | `<thingClass>Example17.MyBuilding</thingClass>` | DLL 必须已被选中并成功加载 |
| 嵌套对象 | `<building><isInert>true</isInert></building>` | 其子节点映射嵌套类型字段 |
| 普通列表 | `<thingCategories><li>ResourcesRaw</li></thingCategories>` | 每项通常是 `li` |
| 多态列表 | `<comps><li Class="Example17.CompProperties_Counter" /></comps>` | Class 指向 properties 类，不是 ThingComp |
| IntVec2 / IntVec3 / Vector 等 | `<size>(1,1)</size>`、`<interactionCellOffset>(0,0,-1)</interactionCellOffset>` | 维数与目标类型对应；不能把地图平面误当 x/y |
| 自定义成对项 | `<costList><Steel>5</Steel></costList>` | 这是 ThingDefCountClass 的自定义格式 |
| StatModifier 列表 | `<statBases><Mass>0.2</Mass></statBases>` | 名称是 StatDef；不是任意字典 |
| 范围或特殊结构 | 参考同一字段的原版例子 | IntRange、FloatRange、flags、字典、曲线都不能凭直觉套格式 |

大多数自定义数据类使用可加载字段和无参构造。属性、任意构造参数、仅 getter 的集合不会自动成为 XML schema。类型若实现约定方法 `LoadDataFromXmlCustom(XmlNode)`，格式由它决定；例如 costList 和 statBases 的简写不能推广到所有 `List<T>`。查到旧加载器行为时，要同时检查 1.7 的 `DirectXmlToObjectNew` / `XmlToObjectUtils`。

## 继承规则

继承发生在 XML Patch 之后、Def 反序列化之前。`ParentName` 指向 `Name`，不能直接指向某个只有 defName 的原版物品。

- 标量字段由子节点覆盖；嵌套结构按节点继续合并。
- `li` 列表通常追加；给子模板写一个新的 `<comps>` 不会自然删除父 comps。
- `<comps Inherit="False">...</comps>` 替换该子树；空子树可以清掉继承内容，但要保留对象正常工作所需的 Comp。
- `statBases`、`costList` 等命名项存在特殊加载格式，继承时按 XML 节点规则合并。改同一 Stat 常可覆盖同名节点；要整体替换才加 `Inherit="False"`。
- 同一个对象块里重复写两次非列表字段会造成重复节点问题。不要用追加第二个 `<label>` 或第二个 `<graphicData>` 来实现覆盖。
- 模板跨 mod 可受加载顺序影响。不要定义与原版或其他 mod 一样的 `Name` 来“偷偷覆盖”；修改模板用显式 Patch 并检查影响的全部子 Def。

完整父链只需追到最接近的少数模板，不必读取整个 Def 目录。原版例子：`BuildingBase → FurnitureBase → FurnitureWithQualityBase`。

## 常用 Def 的字段与组合

这张表是编辑入口，不是“必须全部填写”的 schema。字段的具体语义继续看同类原版 Def；自定义字段只存在于对应的 C# 数据类型中。

| 任务 / Def | 常用字段 | 首选原版目录 |
|---|---|---|
| 物品 `ThingDef` | category、thingClass、graphicData、stackLimit、statBases、thingCategories、comps、tradeability | `ThingDefs_Items` |
| 建筑 `ThingDef` | size、passability、designationCategory、costList/stuffCategories、statBases、building、comps、researchPrerequisites、tickerType | `ThingDefs_Buildings` |
| 材料 `ThingDef` | stuffProps、statBases、thingCategories、graphicData | `ThingDefs_Items/Items_Resource_Stuff.xml` |
| 远程武器 `ThingDef` | verbs、tools、weaponTags、statBases、comps；verbs 关联 projectile | `ThingDefs_Misc/Weapons` |
| 弹体 `ThingDef` | projectile.damageDef、damageAmountBase、speed 等 | `ThingDefs_Misc` 中搜目标 projectile defName |
| 服装 `ThingDef` | apparel.layers、bodyPartGroups、tags、wornGraphicPath、statBases | `ThingDefs_Misc` 中搜同类服装 |
| 动物 / 种族 `ThingDef` | race、statBases、comps、tools；生成配置另用 PawnKindDef | `ThingDefs_Races` |
| `PawnKindDef` | race、combatPower、defaultFactionType、lifeStages 等 | `PawnKindDefs` |
| `RecipeDef` | ingredients、fixedIngredientFilter、products、workAmount、workSpeedStat、workSkill、recipeUsers、workerClass | `RecipeDefs` |
| `ResearchProjectDef` | baseCost、prerequisites、techLevel、researchViewX/researchViewY | `ResearchProjectDefs` |
| `HediffDef` | hediffClass、comps、stages、initialSeverity 等 | `HediffDefs` |
| `ThoughtDef` | thoughtClass / workerClass、stages、durationDays 等 | `ThoughtDefs` |
| `JobDef` | driverClass、reportString、作业行为选项 | `JobDefs` |
| `WorkGiverDef` | giverClass、workType、priorityInType、verb、gerund | `WorkGiverDefs` |
| `StatDef` | workerClass、defaultBaseValue、parts、显示格式及类别 | `StatDefs` |
| `IncidentDef` | workerClass、category、targetTags、points 等 | `IncidentDefs` |
| DLC `GeneDef` / `AbilityDef` 等 | 从对应 DLC 示例和类型字段入手 | `Data/<DLC>/Defs` |

不要用 `GetNamed` 成功与否判断 DLC 依赖完整；自定义类本身引用可选程序集也可能在反射取类型时失败。DLC 可选内容放条件目录或使用确实支持的 MayRequire 节点；无 DLC 的测试同样要通过。

## Recipe 的关键区别

```xml
<RecipeDef>
  <defName>EX17_RecycleSteel</defName>
  <label>recycle steel</label>
  <description>Example recipe for studying the data format.</description>
  <jobString>Recycling steel.</jobString>
  <workAmount>200</workAmount>
  <workSpeedStat>GeneralLaborSpeed</workSpeedStat>
  <workSkill>Crafting</workSkill>
  <ingredients>
    <li>
      <filter><thingDefs><li>Steel</li></thingDefs></filter>
      <count>2</count>
    </li>
  </ingredients>
  <fixedIngredientFilter><thingDefs><li>Steel</li></thingDefs></fixedIngredientFilter>
  <products><Steel>1</Steel></products>
  <recipeUsers><li>CraftingSpot</li></recipeUsers>
</RecipeDef>
```

这是格式例子，不是推荐平衡配方。`ingredients` 表示各组材料需求，`fixedIngredientFilter` 限制可选材料；过滤器和数量一起决定实际消耗。不要假设所有 Recipe 的 `count` 都是物品堆数量：食物配方可以按营养值计量，自定义 ingredientValueGetter 还可能改变它。

把 Recipe 挂到 `recipeUsers` 或工作台 `recipes` 后，还需要可执行该工作台的工作系统、有效工作位置和材料。新增 Recipe 通常不需要新建 JobDriver；手术等特殊行为才需要 worker。

## 追加 C# 数据

```xml
<modExtensions>
  <li Class="Example17.ExampleExtension">
    <cooldownTicks>600</cooldownTicks>
  </li>
</modExtensions>
```

对应类继承 `Verse.DefModExtension`、公开 `int cooldownTicks = 600;`，读取用 `def.GetModExtension<ExampleExtension>()` 并处理返回 null。运行时可变进度存到 Comp/Component，不修改这个共享配置。

完整 ThingComp 与 Def 配套见 [recipes.md](recipes.md)；只改已有定义见 [xml-patches.md](xml-patches.md)。

## 从 1.0 迁移到当前 1.7（2026-10-17 局部核实）

在 `1.7.4981 rev136` 的 Lost Forest / ResourcePackCore 合并迁移中，已核实：

- 动物的 `race/wildness` 改为 `statBases/Wildness`，参考 Core 的 Cat。
- 旧 `statBases/ToxicSensitivity=s` 需换算为 `ToxicResistance=1-s`。附加 Hediff 的 `victimSeverityScaling` 改为 ToxicResistance 时，同时设置 `inverseStatScaling=true`；不要只换名称。此换算针对基础值，不可直接照搬到乘数或偏移量。
- `smeltable` 属于 ThingDef，不属于 `stuffProps`；`soundImpactStuff` 对应当前 `soundImpactBullet`。
- 地形的 `holdSnow` 对应 `holdSnowOrSand`；不接纳污物使用 `filthAcceptanceMask/Inherit=False` 的 `li=None`，参考 WaterBase。
- 炮塔顶部材质由 `building.turretGunDef.graphicData.texPath` 生成。旧 `turretTopGraphicPath` 不再有效；不补 gun 的 graphicData 会在延后解析时空引用。
- `explosionCellMote=Mote_BlastDry` 等旧引用可按当前原版例子迁移到 `explosionCellFleck=BlastDry`；WorkGiver 的同类迁移使用 `forceFleck`。先核对具体 FleckDef。
- `CompProperties_RoomIdentifier` 已不存在；床铺仍需当前 BedBase 所用的 `CompProperties_AssignableToPawn/compClass=CompAssignableToPawn_Bed`。只删除旧 Comp 而不检查当前床铺约定，会留下绘制和归属错误。

声音的 `AudioGrain_Folder/clipFolderPath` 指目录，不代表文件名前缀。若旧包把多个 wav 平放在 Sounds 下，应为真实文件列出 `AudioGrain_Clip/clipPath`，而不是保留无法解析的伪目录。

上述结论的程序集、原版例子及实测入口见 [sources.md](sources.md) 的 Lost Forest 迁移记录。
