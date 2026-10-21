# RimWorld 1.7 性能优化检查入口

基准为 `1.7.4979 rev109`，核对日期 `2026-10-15`，主 DLL 指纹见 [sources.md](sources.md)。以下来自当前本地程序集的定向检查，用于筛选旧版优化方案；没有运行基准或收益结论。

## 已核实的行为

| 主题 | 当前实现 | 对优化设计的影响 |
| --- | --- | --- |
| 尸体内容 | `Corpse.ShouldTickContents=false`；`Thing.DoTick` 在进入内容前检查该接口属性 | 不再为原版常规路径添加旧尸体内部 Pawn Tick 屏蔽 |
| 累计 Tick | `Thing.DoTick` 调用 `TickInterval(tickDelta)` 后清零，`ThingWithComps.TickInterval` 将 delta 传给 Comp | Prefix 跳过并不会保留这次累计时间，不能直接复制建筑降频方案 |
| 组件查询 | `ThingWithComps.GetComp<T>` 有少量组件短路、类型索引和组件数/末项变更检测，部分情况仍扫描 | 先测残余热点，不假设每次全量线性扫描 |
| 建筑查询 | `ListerBuildings.AllBuildingsColonistOfDef` 每次扫描殖民者建筑并返回共享结果缓冲区 | 索引候选需保留顺序、所属阵营和返回集合语义 |
| 建筑跟踪 | `ListerBuildings.Track` 创建 `TrackingScope`，注册后追踪增删，构造过程不填充已有建筑 | 要自行初始化并管理 Dispose；HashSet 不能直接替代有顺序的查询结果 |
| 建造 | `JobDriver_ConstructFinishFrame.MakeNewToils` 使用 tickIntervalAction(delta)，并设置 CanConstruct 失败条件 | 重新识别闭包/IL，验证降频与任务续接分别评估 |
| 装备属性 | `StatPart_GearStatOffset.TransformValue` 逐件计算属性及 Gear offset | 缓存要区分 StatPart 配置，失效不能只靠穿脱 |
| 意识形态 | `Ideo.IdeoTick` 同时运行 Precept Tick 与无效信徒缓存重建 | 跳过整个入口会跳过缓存维护，地图计数不足以覆盖世界对象 |
| 深钻 | `CompDeepDrill.CanDrillNow` 检查电力、基础资源和贵重资源；产出会改变资源格 | 保留无 PowerTrader 语义，处理相邻钻机和地形变化 |

`TrackingScope` 是变更跟踪接口，不是通用完整建筑索引。`ThingOwner` 搜索有泛型和非泛型同名结果，应检查实际成员列表再反编译；容器的 `DoTick` 本身仍能被直接调用，尸体结论限定于原版常规链路。

## 缓存范围补充

2026-10-16 在同一主程序集上核实以下差异，完整依赖、键和失效条件见 [缓存专项核对](../Mods/RimWorldPerformance17/Research/cache-audit.md)。

- `ThingDef.GetCompProperties<T>`、`HediffDef.CompProps<T>` / `CompPropsFor`、`Def.GetModExtension<T>`、Map/Game/World 的 GetComponent、`HediffWithComps.GetComp<T>` 仍扫描列表。它们与 ThingWithComps 的类型索引不是同一个实现。可以研究配置代次或拥有者生命周期内的局部缓存，保留首个匹配与继承语义。
- `StatWorker_MarketValue.CalculableRecipe` 仍扫描配方数据库，可缓存静态配方选择；不等同于缓存最终市值。
- `StatWorker.GetValue(Thing, bool, int)` 有 immutable 缓存与可选临时缓存，`GetValue(StatRequest, bool)` 仍计算并最终处理。不要全局叠加缓存，也不要认为所有 Stat 调用都已缓存。
- `Frame.WorkToBuild` 使用实体 Def 与 Stuff 计算抽象属性。原版 WorkToBuild XML 没有自有 parts，但 FinalizeValue 还乘 Scenario 因子；可在核验最终配置后按 Def、Stuff 和游戏/配置代次研究缓存。
- `Room.GetStat` 已使用 statsAndRoleDirty 和结果字典；可达性有 District 对及 TraverseParms 缓存；`GenConstruct.CanGetResources_NewTemp` 有按 tick 清理且带材料需求哈希等输入的缓存。优先复用这些机制。
- `GenConstruct.CanConstruct` 还设置 JobFailReason 并调用 HistoryEvent 的 Notify_PawnAboutToDo_Job；`StatWorker.StatOffsetFromGear` 处理 Bladelink 和 StatPart。二者不能按“只返回布尔/数值”的纯函数处理。按 tick/帧过期不构成普遍等价保证。

## 工程与验证

2026-10-16 补充核实：`GenThreading.ParallelForEach` 将批次提交 ThreadPool，加载引用解析存在调用；`PathFinder.PathFinderTick` / `ScheduleBatchedPathJobs` / `ForceCompleteScheduledJobs` 已有网格与路径 Job 的依赖、调度和集中完成流程；`TickList.Tick` 本身仍顺序调用实体 DoTick。不能把原版描述为全部串行，也不能推断整个 Pawn/地图可安全并行。实际并行度和收益尚未运行测量。详见 [多线程调研](../Mods/RimWorldPerformance17/Research/multithreading.md)。

新项目使用 net10.0 和根目录主程序集；Harmony 选择共同的 1.7 Harmony 3。不要迁移旧 Unity Jobs/Burst 原生二进制或未经核实的 Prepatcher 改写。缓存与调度先定义依赖、生命周期、最大响应延迟和原版回退路径。

性能测量分开报告模拟 tick 成本、UI 帧耗时、加载耗时与分配。编译通过只能证明编译阶段正确；等价算法、近似缓存和降低工作频率要分别标注，不能仅以 TPS 变高证明行为正确。

本次十二个指定参考 MOD 的详细分析、优先级和证据指纹位于 [RimWorldPerformance17 调研目录](../Mods/RimWorldPerformance17/Research/README.md)。其中工坊 `3697920753` 的 Architect Menu Optimizer 本地副本已声明并提供 1.7 DLL，其 `EngineFrame.Count` 使用 Waylith.Time.FrameCount；这不是游戏内兼容性验证。
