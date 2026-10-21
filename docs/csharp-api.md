# C# API：按开发任务查入口

所有游戏类型以当前根目录 `Assembly-CSharp.dll` 为准。以下给出职责和容易踩错的契约；更多准确签名见 [api-signatures.md](api-signatures.md)。游戏没有承诺稳定的第三方 SDK，1.7 后续 build 的私有实现仍可能变化。

## 定义与配置

| 入口 | 用法 / 约束 |
|---|---|
| `Verse.DefDatabase<T>.GetNamed(name, errorOnFail)` | 必须存在的 Def；加载时机要晚于数据库建立 |
| `DefDatabase<T>.GetNamedSilentFail(name)` | 可选 Def，返回 null 时正常处理 |
| `DefDatabase<T>.AllDefs` / `AllDefsListForReading` | 遍历已加载定义；后者当只读集合，不随意增删 |
| `Verse.Def.GetModExtension<T>()` | 读取 `modExtensions` 中配置；可以为 null |
| `[RimWorld.DefOf]` | 将静态 Def 字段按名字绑定；类型和 defName 要一致 |
| `RimWorld.DefOfHelper.EnsureInitializedInCtor(typeof(...))` | DefOf 类标准静态构造写法；不能把构造期业务初始化当作已完成绑定 |
| `Verse.LoadedModManager.RunningModsListForReading` | 当前启用内容包，区别于安装但未启用的 mod |
| `Verse.ModLister` | 包存在/激活状态查询；选实际 packageId 对应方法，勿把显示名和 ID 混用 |

DefOf 示例见 [recipes.md](recipes.md)。可选 DLC 的字段绑定应使用原版提供的条件属性/条件内容方案，不能仅靠 null 判断绕过所有加载错误。

## Thing、地图与生成

| 入口 | 用法 / 约束 |
|---|---|
| `ThingMaker.MakeThing(ThingDef, ThingDef stuff)` | 创建实体，尚未放进地图；可由 stuff 制作的 Def 必须提供有效材料 |
| `GenSpawn.Spawn(...)` / `TrySpawn(...)` | 放入地图，可能依据 WipeMode 清除已有内容；先检查格子和规则 |
| `GenPlace.TryPlaceThing(...)` | 附近放置/合堆，检查 bool 结果；不能假定总能找到位置 |
| `thing.SetFaction(...)` | 通过实体 API 设置派系；生成建筑等需要明确所有权 |
| `thing.SplitOff(count)` / 堆叠 API | 用游戏 API 拆分、转移，保持 Comp 和堆叠契约 |
| `thing.Destroy(...)` / `DeSpawn(...)` | 销毁和离开地图有不同含义；触发清理和通知 |
| `ThingWithComps.GetComp<T>()` / `Thing.TryGetComp<T>()` | 取 Comp；后者是扩展方式，依实际类型检查可用性；返回可以为 null |
| `thing.Map` / `Spawned` / `Destroyed` | 地图实体状态，操作前检查 |
| `thing.MapHeld` / `PositionHeld` | 沿持有链查询；可能是容器中物品 |
| `map.listerThings` / `map.mapPawns` | 地图对象索引；优先使用适合的索引和请求，避免每 tick 全图扫描 |
| `map.thingGrid` / `edificeGrid` / `terrainGrid` | 格子级查询；使用当前 1.7 getter/扩展，不能直接写内部格子数组 |

地图坐标是 `IntVec3(x, 0, z)`；`IntVec2` 通常用于尺寸。`LocalTargetInfo` 可包含 Thing 或格子，不保证 `Thing` 非空。世界目标使用 `GlobalTargetInfo` / `PlanetTile` 等，不把世界 tile 当作地图格子或裸整型。

## ThingComp 的覆盖点

```csharp
// 声明在 Verse.ThingComp；不是可独立编译的类。
public virtual void Initialize(CompProperties props);
public virtual void PostPostMake();
public virtual void PostSpawnSetup(bool respawningAfterLoad);
public virtual void PostDeSpawn(Map map, DestroyMode mode = DestroyMode.Vanish);
public virtual void PostDestroy(DestroyMode mode, Map previousMap);
public virtual void PostExposeData();
public virtual void CompTick();
public virtual void CompTickInterval(int delta);
public virtual void CompTickRare();
public virtual void CompTickLong();
public virtual string CompInspectStringExtra();
public virtual IEnumerable<Gizmo> CompGetGizmosExtra();
public virtual void ReceiveCompSignal(string signal);
```

`parent` 为 `ThingWithComps`，`props` 为共享的 `CompProperties`。惯用 `Props => (MyCompProperties)props`。properties 构造中设置 `compClass = typeof(MyComp)`；XML 的 Class 指向 properties。需要 ticking 的对象必须选适当 `tickerType`，其 delta 规则见 [architecture.md](architecture.md)。

自定义 properties 可以覆盖 `ConfigErrors(ThingDef parentDef)` 提供加载期错误，发现无效配置比运行时不断报错更便宜。实体合堆/拆堆需要考虑 `AllowStackWith`、`PreAbsorbStack`、`PostSplitOff` 等状态一致性；复杂持久化物品不要直接允许堆叠。

## 工作系统：WorkGiver → Job → JobDriver → Toil

1. `WorkGiverDef` 指向 `RimWorld.WorkGiver` / `WorkGiver_Scanner` 子类，并归属于 `WorkTypeDef`。
2. scanner 从潜在目标选出可执行对象；`HasJobOnThing(Pawn, Thing, bool forced)` 应便宜、无副作用，不能预扣材料。
3. `JobOnThing(...)` 返回 `Verse.JobMaker.MakeJob(jobDef, target)`；`JobMaker` 在 **Verse**。
4. `JobDef.driverClass` 指向 `Verse.AI.JobDriver` 子类。
5. Driver 的 `TryMakePreToilReservations(bool)` 预订资源，`MakeNewToils()` 产出行为步骤。
6. Toil 通过 initAction、tickAction / tickIntervalAction、结束条件和 finishAction 执行。工作中目标随时可能失效。

需要的检查通常包括：目标仍存在/Spawned、未禁止、可预订、可到达、Pawn 能执行工作、材料/电力/燃料等仍有效。`forced` 代表玩家强制操作语境，不等于可以无视生命状态、可达性或资源存在。

`pawn.CanReserve`、`pawn.Reserve`、`Toils_Goto.GotoThing`、`Toils_General.Wait`、Toil 的 FailOn 扩展是常见入口；确认具体重载后使用。清理在 job 中断、失败或成功时都要正确执行。不要在 `MakeNewToils` 的枚举构建阶段完成真正的产出或消耗；应放在会执行的 Toil 动作中。

最小 Driver（没有目标，因此不需要预订；仅演示链路）：

```csharp
using System.Collections.Generic;
using Verse.AI;

namespace Example17;

public sealed class JobDriver_ExampleWait : JobDriver
{
    public override bool TryMakePreToilReservations(bool errorOnFailed) => true;
    protected override IEnumerable<Toil> MakeNewToils()
    {
        yield return Toils_General.Wait(60);
    }
}
```

配套 XML 为 `<JobDef><defName>EX17_Wait</defName><driverClass>Example17.JobDriver_ExampleWait</driverClass><reportString>waiting.</reportString></JobDef>`，置于 `<Defs>` 内。它只定义行为，仍需 WorkGiver、其他 JobGiver 或调试入口分派。生产配方优先复用原版做账单的工作链。

多地图连续任务：`Pawn_JobTracker.StopAll` 会清排队 Job，`EndCurrentJob` 默认可重新分派，`JobDriver.IsContinuation` 基类默认返回 true。队列、`resumeCurJobAfterwards` 或相同 JobDef 都不构成跨图目标保持协议。需求选食物、找床和娱乐还分别经过 JobGiver/utility，不能只扩 WorkGiver。当前 1.7 证据与建议接法见 [连续跨层寻路与任务协调](multilayer-routing.md)；协议尚未实现或游戏内验证。

## Pawn、健康与心情

| 任务 | 入口 / 注意事项 |
|---|---|
| 创建 Pawn | `Verse.PawnGenerator`、`PawnGenerationRequest`；复杂 request 不猜构造参数，选当前重载 |
| 增加健康状态 | `HediffMaker.MakeHediff`、`pawn.health.AddHediff`；指定正确 BodyPartRecord |
| 移除健康状态 | `pawn.health.RemoveHediff`，保留通知链 |
| Hediff 行为 | `HediffComp.CompPostPostAdd(DamageInfo?)`、`CompPostTick(ref float)`、`CompPostTickInterval(ref float, int delta)`、`CompPostPostRemoved`、`CompExposeData` |
| 查询已有状态 | `pawn.health.hediffSet` 及其查询方法；不要直接从内部列表 Remove |
| 给予记忆 | `pawn.needs.mood?.thoughts.memories` 对应 API；动物/特殊 Pawn 可能没有 mood 或其他 tracker |
| 情境思想 | `RimWorld.ThoughtWorker.CurrentStateInternal(Pawn)`；返回 ThoughtState，查询中不改变 Pawn |
| 需求/能力/DLC 基因 | 对应 Def + tracker / worker；先查对应 DLC 原版例子 |

修改 `Severity` 是健康状态参数，不等于直接改变生命值；不同 hediff 的阶段、消失条件和免疫逻辑不同。添加 body-part hediff 必须使用该 Pawn 身体树中的记录，不能拿另一 Pawn 的 BodyPartRecord。

## 数值、事件与配方 worker

| 类型 | 典型覆盖点 | 约束 |
|---|---|---|
| `RimWorld.StatWorker` | `GetValueUnfinalized(StatRequest, bool)`、`FinalizeValue(...)` | 先了解 statBases、stuff、品质、StatPart 与最终值流程，避免重复乘系数 |
| `RimWorld.ThoughtWorker` | protected `CurrentStateInternal(Pawn)` | 便宜且无副作用；常被频繁查询 |
| `RimWorld.IncidentWorker` | protected `CanFireNowSub(IncidentParms)`、`TryExecuteWorker(IncidentParms)` | 目标不总是 Map；TryExecuteWorker 返回是否真正执行成功 |
| `Verse.RecipeWorker` | `ApplyOnPawn(...)`、`ConsumeIngredient(...)`、`Notify_IterationCompleted(...)` | RecipeWorker 在 Verse；普通生产数量优先 XML |
| `Verse.PlaceWorker` | 放置检验与预览覆盖点 | 先查具体签名，不从旧版复制；预览查询避免副作用 |

Worker 通常由 Def 持有、缓存并复用。不要用 worker 字段保存“当前 Pawn”“当前事件目标”等每次调用状态。

## UI、设置与 Gizmo

- `Verse.Mod`：`GetSettings<T>()`、`SettingsCategory()`、`DoSettingsWindowContents(UnityEngine.Rect)`、`WriteSettings()`。一个 Mod 实例通过 GetSettings 管理同一个设置类型。
- `Verse.ModSettings`：`ExposeData()` + Scribe；不是 GameComponent。完整例子见 [recipes.md](recipes.md)。
- `Verse.Window`：覆盖 `DoWindowContents(Rect)`；通过 `Find.WindowStack.Add(window)` 显示，避免在每帧反复创建窗口。
- `Verse.Listing_Standard` / `Widgets`：IMGUI 风格布局与控件。每次 Begin/End 配对，不在 OnGUI 里重复生成世界内容。
- `Command_Action` / `Command_Toggle`：通过 ThingComp 的 `CompGetGizmosExtra()` 返回。action 才执行业务；标签与图标获取只用于显示。
- 显示文本使用 `"EX17_Key".Translate(...)`；选中对象的 Map 用 parent.Map，避免跟随玩家切换地图而操作错图。

## 生命周期速查

| 类 | 构造 / 取用 | 常见回调 |
|---|---|---|
| MapComponent | `.ctor(Map)`，`map.GetComponent<T>()` | MapGenerated、FinalizeInit、MapComponentTick、MapRemoved |
| WorldComponent | `.ctor(World)`，`world.GetComponent<T>()` | FinalizeInit(bool fromLoad)、WorldComponentTick |
| GameComponent | 子类 `.ctor(Game)`，`Current.Game.GetComponent<T>()` | StartedNewGame、LoadedGame、FinalizeInit、GameComponentTick |
| ThingComp | properties 指定 compClass | PostPostMake、PostSpawnSetup、PostDeSpawn、PostExposeData |
| HediffComp | Hediff properties 指定 compClass | CompPostPostAdd、CompPostPostRemoved、CompExposeData |

## 扩展检索地图

渲染：`PawnRenderTree`、`PawnRenderNode`、`PawnRenderNodeWorker`；战斗：`Verb`、`VerbProperties`、`Projectile`、`DamageWorker`；世界：`WorldObject`、`Caravan`、`PlanetTile`；群体 AI：`Verse.AI.Group.Lord` / LordJob；地图生成：GenStep / GenStepDef；任务：`RimWorld.QuestGen` 的 QuestNode。

这些领域的复杂覆写只给搜索入口，不声称可凭名字直接实现。先在原版 Def 定位 worker/节点类型，再读目标方法及少量调用者。若需求只涉及上表中的现成扩展点，无需探索这些系统。

## 旧版覆写可能编译通过却不再被调用

当前 1.7.4981 定向核实的签名：

```csharp
// Verse.PlaceWorker
public virtual AcceptanceReport AllowsPlacing(BuildableDef checkingDef, IntVec3 loc, Rot4 rot,
    Map map, Thing thingToIgnore = null, Thing thing = null);
public virtual void DrawGhost(ThingDef def, IntVec3 center, Rot4 rot, UnityEngine.Color ghostCol,
    Thing thing = null);
// Verse.ThingComp
public virtual void PostDeSpawn(Map map, DestroyMode mode = DestroyMode.Vanish);
```

从旧 DLL 恢复源码时，反编译器可能把过期 override 显示为新的 virtual 方法。必须对照当前基类签名恢复 override，否则编译通过也不会执行。旧 `Thing.Draw()` 的实体自定义绘制应迁移到当前 `protected override DrawAt(Vector3 drawLoc, bool flip=false)`；必要时保留相应绘制阶段处理。

`JobDriver.DriverTick()` 仍每 tick 调用 `Toil.tickAction`，`DriverTickInterval(delta)` 另外调用 `tickIntervalAction(delta)`。只因出现 interval API 就改名，会改变按 tick 累积的工作量；若选择 interval 路径，需把时间累计、经验和舒适度等一并按 delta 处理。来源见 [sources.md](sources.md) 的 2026-10-17 迁移记录。
