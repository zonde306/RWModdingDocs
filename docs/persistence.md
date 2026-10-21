# 状态归属、Scribe 与存档兼容

## 先确定状态属于谁

| 数据 | 正确归属 | 持久化入口 |
|---|---|---|
| 共享配置、基础数值 | Def / CompProperties / DefModExtension | 由 mod XML 重新加载，不保存成实例副本 |
| 单个 Thing 的进度、开关 | ThingComp / 实体 | `PostExposeData` / `ExposeData` |
| 单个 Hediff 的状态 | HediffComp | `CompExposeData` |
| 当前 Job 的可恢复进度 | JobDriver / Job 数据 | `ExposeData`，并能重建 Toil |
| 单图数据 | MapComponent | `ExposeData` |
| 世界数据 | WorldComponent | `ExposeData` |
| 整局数据 | GameComponent | `ExposeData` |
| 玩家全局偏好 | ModSettings | `ExposeData`，写到 Config 而非存档 |
| 可重建索引/资源缓存 | 生命周期受控的缓存 | 不保存；读档完成后重建 |

## Scribe 类型选择

```csharp
using System.Collections.Generic;
using Verse;

namespace Example17;

public sealed class SavedState : IExposable
{
    private int elapsedTicks;
    private bool enabled = true;
    private Thing target;
    private ThingDef product;
    private List<int> samples = new List<int>();

    public void ExposeData()
    {
        Scribe_Values.Look(ref elapsedTicks, "elapsedTicks", 0);
        Scribe_Values.Look(ref enabled, "enabled", true);
        Scribe_References.Look(ref target, "target");
        Scribe_Defs.Look(ref product, "product");
        Scribe_Collections.Look(ref samples, "samples", LookMode.Value);
        if (Scribe.mode == LoadSaveMode.PostLoadInit)
            samples ??= new List<int>();
    }
}
```

| API | 用途 | 常见误用 |
|---|---|---|
| `Scribe_Values.Look` | bool/int/float/string/enum 等值 | 错改默认值，导致旧档缺字段时行为变化 |
| `Scribe_Defs.Look` | Def 引用，按定义标识恢复 | 用 Deep 复制共享 Def |
| `Scribe_References.Look` | 游戏中已有的可引用实体 | 在 LoadingVars 阶段立即解引用；目标可能尚未解析 |
| `Scribe_Deep.Look` | 由当前持有者拥有的 IExposable 对象 | 对同一个 Thing 在多个地方深存，造成重复实体 |
| `Scribe_Collections.Look` | List/Dictionary 等 | 猜 LookMode；遗漏字典工作列表或 null 重建 |

集合中的 `LookMode.Value` / `Def` / `Reference` / `Deep` 对应上表概念。不同泛型重载、字典 key/value 模式和读引用时需要的 working lists，先查 [api-signatures.md](api-signatures.md) 或精确重载；不要把 List 的调用机械地套到 Dictionary。

## 读写阶段

`Scribe.mode` 常见值：`Saving`、`LoadingVars`、`ResolvingCrossRefs`、`PostLoadInit`、`Inactive`。`ExposeData` 在不同模式下被重复调用，不是每次只执行一次的初始化回调。

- `Saving`：写入当前状态，避免在这里推进模拟或发奖励。
- `LoadingVars`：读基础值和自有对象。引用字段未必已可用。
- `ResolvingCrossRefs`：恢复已有对象引用；不能假设所有业务缓存已重建。
- `PostLoadInit`：验证数据、补默认集合、重建可由已加载对象恢复的状态。
- 构造函数可能在读档时调用：只做便宜、局部、无副作用的初值设置，不访问地图、生成实体或安装补丁。

重写带有基类实现的方法时调用 `base.ExposeData()` / `base.PostExposeData()` 等；不要因为当前基类看起来为空就省略。自定义对象的 Deep 读取需要合适的构造方案和稳定的类型身份。

## 自动发现的全局组件

已核实的构造/生命周期差异：

```csharp
using RimWorld.Planet;
using Verse;

namespace Example17;

public sealed class ExampleMapComponent : MapComponent
{
    public ExampleMapComponent(Map map) : base(map) { }
    public override void FinalizeInit() { base.FinalizeInit(); }
}

public sealed class ExampleWorldComponent : WorldComponent
{
    public ExampleWorldComponent(World world) : base(world) { }
    public override void FinalizeInit(bool fromLoad) { base.FinalizeInit(fromLoad); }
}

public sealed class ExampleGameComponent : GameComponent
{
    // Game.FillComponents 用 Activator.CreateInstance(type, game) 创建它。
    // GameComponent 本身没有带 Game 的基类构造函数。
    public ExampleGameComponent(Game game) { }
    public override void StartedNewGame() { base.StartedNewGame(); }
    public override void LoadedGame() { base.LoadedGame(); }
}
```

一般由游戏发现具体子类并创建，不需要给 Game/Map/World 再随手 Harmony 一个重复实例。访问分别用 `map.GetComponent<T>()`、`world.GetComponent<T>()`、`Current.Game.GetComponent<T>()`；调用前确认所属对象存在。

`MapComponent.FinalizeInit()` 与 `WorldComponent.FinalizeInit(bool fromLoad)` 签名不同。新地图生成的初始化、读档后的初始化和新会话初始化也不等价；需要哪条路径，就实现对应回调并确保幂等。

## 兼容性规则

- defName、程序集名、namespace/type 的身份和 Scribe label 是持久化契约。重构前检查旧存档是否保存这些名字。
- 新字段给合理默认值；旧档没字段时应自然可用。必要时保存 `dataVersion`，只迁移旧版本，并使迁移可重复执行。
- 改 label 时显式保留迁移读取逻辑，不能简单换字符串。不要在保存模式同时写两个同名节点。
- 目标被销毁、另一个 mod 被移除、地图被卸载后，引用可能为空或已无效。读档后做有业务意义的验证，不用 catch-all 隐藏数据丢失。
- 不直接用 .NET BinaryFormatter 作为新存档格式。当前运行时兼容开关不构成新 mod 的持久化设计。
- 移除 mod 是否安全取决于它是否留下自定义实体、Hediff、组件等；只改数值和新增存档类型的风险不同，交付说明应准确描述。

验证至少覆盖：新建对象 → 改变状态 → 保存 → 退出/重启 → 读档 → 再次保存；增加字段时用缺少该字段的旧档副本测试。不要用玩家原档做破坏性试验。

## 原版地图压缩与实体对照（1.7.4980，2026-10-16）

已定向读取 Map.ExposeData / FinalizeLoading、MapFileCompressor.BuildCompressedString / ThingsToSpawnAfterLoad 和 CompressibilityDeciderUtility.IsSaveCompressible：

- 地图对可压缩物体按格保存定义短哈希，加载时 ThingMaker.MakeThing 重建。因此部分石块等物体的 ThingID 变化属于原版语义；持久实体对照应区分精确引用身份和压缩物体的类型/位置/数量等事实。
- IsSaveCompressible 还依赖当前正在保存的地图 compressor 和引用判定。不能在任意 tick 中调用后把结果当作稳定的 Def 配置；脱离保存上下文可能记录错误。
- FinalizeLoading 先生成非建筑，再生成建筑，使用 FullRefund。GenSpawn.SpawningWipes / WipeAndRefundExistingThings 会清除部分阻挡物品和建筑下重叠的污渍；新图快照与读档快照的逐 ThingID/数量对照必须单独解释这些原版清理，不能将所有差异都放过。
- Settlement.ShouldRemoveMapNow 会在没有玩家驻留 Pawn、家园或其他保留条件时允许卸载地图。验证多图持久化时应准备具备合法保留条件的测试图，而不是更改原版卸载政策。

以上行为已在 [RimLayers M1 独立生成/重启验收](../Mods/RimLayers/Docs/M1-VALIDATION.md)中核对；只证明该版本、该场景，不代表原版所有物体或第三方保存策略都相同。


## 1.7.4981：加载旧地图时改变存储尺寸的入口

2026-10-18 定向核实 `Map.ExposeData`：LoadingVars 先读取 MapInfo，再 `ConstructComponents`，随后读取原版网格及 MapComponent。因此需要追加坐标带时，必须在这次原生分配之前准备新尺寸；到 FinalizeInit 才改 Size 会使原生缓冲区与索引不一致。

RimLayers 0.1.1 的受限迁移保持宽度不变，使所有原地面 cell index 仍为新缓冲区前缀；只改本次 Scribe 内存 XML，不覆写原文件。typed grid 入口按原来的每格字节数补齐数据，恢复后填上层空域及隔离带，带自身尺寸的 BoolGrid/ByteGrid 等另行修复。原版实体/Job/预订仍由原版加载。该方法不等于任意第三方网格都可按长度扩容。

已核实的细节：

- `MapSerializeUtility.LoadUshort` 使用 `map.info.NumCells` 调用 `DataSerializeUtility.LoadUshort`，解码为每格两个连续字节；int/uint 各四字节。新尺寸与旧数组必须在进入该循环前对齐。
- TerrainGrid 的 top/under/foundation/temp/color 由原版序列化；新增 topGrid 格应有明确的空域/隔离 Def 短哈希，不能以零哈希默许 fallback Soil。
- `DataExposeUtility.LookBitArray` 使用按位存储；长度不匹配还涉及旧六位格式兼容，不能让新 NumCells 与原八位字节数不匹配后直接调用。原地图位前缀需保持，新字节长度为 `ceil(newCells/8)`。
- `MapFileCompressor` 的 `compressedThingMap` 同样会在后续解码中按新 NumCells 读取，不能只补地形与屋顶而遗漏压缩实体图。
- WaterInfo.ExposeData 的外层分支虽未限定 LoadingVars，但内部 `Scribe_Values.Look<string>` **只在 LoadingVars 赋值**；其他读档阶段局部字符串保持 null，不重新解码覆盖流场。应核对完整调用链，不添加重复阶段补丁。河流为 x-major `(x * height + z) * 2` 的两个 float 通道；保持宽度但增加高度时仍须逐列重排，Map.LoadingVars 末尾一次转换即可。

实际 60×60→60×204 的运行、另存再次加载、全部地面网格与持久实体审计见 [0.1.1 更新验收](../Mods/RimLayers/Docs/USABILITY-VALIDATION.md)。核实成员及基准见 [sources](sources.md)。


## 1.7.4982：实体暂存、地图所有权与休眠（2026-10-20）

`Thing.SpawnSetup` 会将实体加入 `map.spawnedThings`：已生成实体的 `holdingOwner` 通常不是 null，而是原版地图容器。判断第三方容器接管时，必须区分此地图所有者与背包、携带、运输容器。`DeSpawn` 负责移除地图容器和各网格登记；不能仅清空字段来转移所有权。`GenSpawn.Spawn` 成功后也应核对地图容器，而不是要求 `holdingOwner == null`。

游戏级 `GameComponent` 可以实现 `IThingHolder` 和 `ISuspendableThingHolder`，以 `ThingOwner<Thing>` 保存原对象，并以 `Scribe_Deep` 保存唯一所有者、以 `Scribe_References` 保存恢复记录中的引用。`IsContentsSuspended=true` 使用原版实体休眠语义；不要调用该容器的模拟 tick 来“补偿”。递归枚举时，一些 `IThingHolder.GetDirectlyHeldThings()` 返回 null，但仍可有子 holder，必须分别处理。

RimLayers 独立原生测试 `v1-settings-recovery-20261020-224405` 验证了暂存 Pawn 的年龄和饥饿值暂停、装备/背包/携带物身份保留，以及退出重启读档、来源地图删除后仍由游戏级容器保全。跨地图释放要求明确确认；该证据不覆盖任意第三方容器的模拟约定。

`Pawn_PathFollower.TryRecoverFromUnwalkablePosition` 在附近无原版可走格时会销毁 Pawn。需要保全异常空域 Pawn 的 mod 应在该分支之前转入自己的唯一恢复所有者，而不改变普通受阻地面的原版处理。`Thing.BeingTransportedOnGravship` 是可直接读取的原生飞船运输标记，应与事务作用域、容器所有权一起排除合法运输。

### 活跃任务 Pawn 暂存时保留携带物（1.7.4982，2026-10-20）

`Pawn.DeSpawn` 在仍有当前任务时调用 `jobs.StopAll()`；`Pawn_JobTracker.CleanupCurrentJob` 默认可能调用 `TryDropCarriedThing`。因此“原 Pawn 放入 ThingOwner”本身不足以保证携带物留在该 Pawn 内。纠错转移应保留原版任务清理和预留释放，仅对正在转移的那个 Pawn 使用 `carryThingAfterJobOverride=true`；不能全局禁止任务掉落。`Notify_Teleported` 因任务失效触发清理的分支也需要同样保护。


### 暂存 Pawn 的存续保护（1.7.4982，2026-10-21）

游戏级暂存容器还应接入 `PawnsFinder` 的全对象与存活对象枚举，并让 `WorldPawnGC.GetCriticalPawnReason` 识别该所有者。只登记存档引用不能替代清理保护。`GameEnder.CheckOrUpdateGameOver` 的保护必须以容器内实际存在、未死亡/销毁的 `IsFreeColonist` 为条件；普通敌对 Pawn 不应保住殖民地。RimLayers 独立原版测试 `v1-settings-recovery-20261021-001625` 直接调用原版入口，在移开其他殖民者后验证了活殖民者的正例和敌对暂存者的反例，并验证实际 GC 返回暂存原因。
