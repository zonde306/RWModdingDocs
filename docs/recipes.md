# 可组合的 1.7 开发示例

本页是自己的最小示例，不直接修改原版。示例代码和 XML 只作为文档保存，**没有安装或启用示例 mod**。复制时替换 namespace、程序集名、packageId、Def/key 前缀。构建和静态检查状态见 [sources.md](sources.md)；游戏内行为仍需按 [debugging.md](debugging.md) 验证。

## 完整示例：带设置与存档的 ThingComp

目录使用 [mod-layout.md](mod-layout.md) 的单版本根目录结构。工程文件使用 [environment.md](environment.md) 的完整 csproj；本例没有 Harmony 依赖。C# 保存为 `Source/Example17/Counter.cs`，编译结果为 `Assemblies/Example17.dll`。

`About/About.xml`：

```xml
<ModMetaData>
  <name>Example 1.7 Counter</name>
  <author>Example Author</author>
  <packageId>example.agentdemo</packageId>
  <supportedVersions><li>1.7</li></supportedVersions>
  <description>A saved ThingComp counter for development.</description>
</ModMetaData>
```

`Source/Example17/Counter.cs`：

```csharp
using System.Collections.Generic;
using UnityEngine;
using Verse;

namespace Example17;

public sealed class ExampleSettings : ModSettings
{
    public bool enableCounting = true;
    public override void ExposeData()
    {
        base.ExposeData();
        Scribe_Values.Look(ref enableCounting, "enableCounting", true);
    }
}

public sealed class ExampleMod : Mod
{
    public readonly ExampleSettings Settings;

    public ExampleMod(ModContentPack content) : base(content)
    {
        Settings = GetSettings<ExampleSettings>();
    }

    public override string SettingsCategory() => Content.Name;

    public override void DoSettingsWindowContents(Rect inRect)
    {
        var listing = new Listing_Standard();
        listing.Begin(inRect);
        listing.CheckboxLabeled("EX17_EnableCounting".Translate(), ref Settings.enableCounting);
        listing.End();
    }
}

public sealed class CompProperties_Counter : CompProperties
{
    public int increment = 1;

    public CompProperties_Counter()
    {
        compClass = typeof(CompCounter);
    }

    public override IEnumerable<string> ConfigErrors(ThingDef parentDef)
    {
        foreach (var error in base.ConfigErrors(parentDef)) yield return error;
        if (increment <= 0) yield return "increment must be positive";
    }
}

public sealed class CompCounter : ThingComp
{
    private int count;
    private CompProperties_Counter Props => (CompProperties_Counter)props;

    public override void CompTickRare()
    {
        base.CompTickRare();
        var mod = LoadedModManager.GetMod<ExampleMod>();
        if (mod != null && !mod.Settings.enableCounting) return;
        count += Props.increment;
    }

    public override string CompInspectStringExtra()
    {
        return "EX17_Count".Translate(count);
    }

    public override void PostExposeData()
    {
        base.PostExposeData();
        Scribe_Values.Look(ref count, "count", 0);
    }
}
```

`Defs/ThingDefs_Buildings/Counter.xml`：

```xml
<Defs>
  <ThingDef ParentName="BuildingBase">
    <defName>EX17_Counter</defName>
    <label>example counter</label>
    <description>A small building with a saved counter.</description>
    <thingClass>Building</thingClass>
    <graphicData>
      <texPath>Things/Building/Production/CraftingSpot</texPath>
      <graphicClass>Graphic_Single</graphicClass>
      <drawSize>(1,1)</drawSize>
    </graphicData>
    <size>(1,1)</size>
    <rotatable>false</rotatable>
    <passability>Standable</passability>
    <altitudeLayer>FloorEmplacement</altitudeLayer>
    <designationCategory>Misc</designationCategory>
    <statBases>
      <MaxHitPoints>100</MaxHitPoints>
      <WorkToBuild>100</WorkToBuild>
      <Mass>1</Mass>
      <Flammability>0</Flammability>
    </statBases>
    <costList><Steel>5</Steel></costList>
    <tickerType>Rare</tickerType>
    <comps>
      <li Class="Example17.CompProperties_Counter"><increment>1</increment></li>
    </comps>
  </ThingDef>
</Defs>
```

它复用原版 CraftingSpot 单图，免去新美术资产。英文/简中 Keyed 与 DefInjected 文件直接使用 [assets-localization.md](assets-localization.md) 的示例。

行为：每次 Rare 回调计数增加 1；这是每 250 模拟 tick 的粗粒度事件计数，不是现实时间计时。每个建筑有自己的 count，设置 enableCounting 为玩家全局设置。没有在 PostSpawnSetup 重置 count，因此读档不会重复初始化。

验证：启用后从“杂项”建造/开发者生成计数器，检查 inspect 文本增长；关闭设置后停止增加；保存、重启、读档后数值保留，再打开设置继续增长。不要用此例的 int 计数器设计无限期高频经济系统。

## DefOf 模式

```csharp
using RimWorld;
using Verse;

namespace Example17;

[DefOf]
public static class ExampleDefOf
{
    public static ThingDef EX17_Counter;

    static ExampleDefOf()
    {
        DefOfHelper.EnsureInitializedInCtor(typeof(ExampleDefOf));
    }
}
```

字段名匹配 defName，字段类型匹配实际 Def 类型。本例要求上方建筑 Def 已加载；可选内容不能复制这个模式后强行当必选 Def 用。不要在 Mod 构造里读这个字段并长期缓存空值。

## 追加配置而不改变 Def 类型

```csharp
using Verse;

namespace Example17;

public sealed class ExampleExtension : DefModExtension
{
    public int cooldownTicks = 600;
}
```

配套 XML 与读取契约见 [xml-defs.md](xml-defs.md)。如果只要共享配置，使用 extension 通常比自定义 ThingDef 子类更容易与其他 mod 共存。

## 其他任务的组合路线

| 目标 | 直接组合 |
|---|---|
| 调整某物品堆叠上限 | [xml-patches.md](xml-patches.md) 的存在性分支，确认字段是否只在父模板 |
| 新配方 | [xml-defs.md](xml-defs.md) 的 Recipe；关联现有工作台，复用原版工作链 |
| 新的 Pawn 操作 | [csharp-api.md](csharp-api.md) 的 JobDriver + JobDef，再补任务分派、预订和失败条件 |
| 单图索引与周期任务 | [persistence.md](persistence.md) 的 MapComponent，注册/撤销实体，读档重建缓存 |
| 原版方法追加行为 | [harmony.md](harmony.md) 的精确 Postfix，按 Def/extension 约束范围 |
| 旧档增加字段 | [persistence.md](persistence.md) 的缺字段默认值、PostLoadInit 与迁移策略 |

这些路线避免重新搜索整个游戏，但不是未验证 API 的通行证；需求离开示例范围时，只核实新增的那一小块接口。
